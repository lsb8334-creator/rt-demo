# rt-demo
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>스포츠 스폰서십 RT 과제 (2AFC + 오디오 조작)</title>
  <!-- jsPsych core & plugins (v7.x) -->
  <script src="https://unpkg.com/jspsych@7.3.3/dist/jspsych.js"></script>
  <script src="https://unpkg.com/@jspsych/plugin-html-keyboard-response@1.1.3"></script>
  <script src="https://unpkg.com/@jspsych/plugin-html-button-response@1.3.0"></script>
  <script src="https://unpkg.com/@jspsych/plugin-preload@1.1.3"></script>
  <script src="https://unpkg.com/@jspsych/plugin-instructions@1.1.3"></script>
  <script src="https://unpkg.com/@jspsych/plugin-survey-text@1.0.3"></script>
  <script src="https://unpkg.com/@jspsych/plugin-survey-likert@1.0.4"></script>
  <script src="https://unpkg.com/@jspsych/plugin-video-button-response@1.3.0"></script>
  <link rel="stylesheet" href="https://unpkg.com/jspsych@7.3.3/dist/jspsych.css" />
  <style>
    body { max-width: 900px; margin: 0 auto; padding: 24px; font-family: system-ui, -apple-system, Segoe UI, Roboto, Noto Sans KR, sans-serif; }
    .stimrow { display:flex; gap:32px; align-items:center; justify-content:center; }
    .stimrow img { width: 220px; height: auto; object-fit: contain; border:1px solid #ddd; padding:12px; }
    .small { color:#666; font-size: 13px; }
    .center { text-align:center; }
    .kbd { display:inline-block; padding:2px 6px; border:1px solid #999; border-bottom-width:2px; border-right-width:2px; border-radius:4px; }
  </style>
</head>
<body>
<div id="jspsych-target"></div>
<script>
// =============================
// 1) 실험 파라미터 (여기만 수정하면 됨)
// =============================
const EXP = {
  // 반응키: 왼쪽/오른쪽
  left_key: 'f',
  right_key: 'j',
  // 시간 제한(ms): 자극 제시 후 응답 허용 시간
  rt_deadline: 800,
  // 고정 ISI / fixation(ms)
  fixation_ms: 500,
  isi_ms: 300,
  // 참가자에게 보여줄 안내문(간단)
  instructions: [
    '이 실험은 경기 하이라이트를 본 뒤, 로고를 빠르게 구분하는 과제입니다.',
    `화면에 두 개의 로고가 나타나면, <span class="kbd">${'f'.toUpperCase()}</span> (왼쪽) 또는 <span class="kbd">${'j'.toUpperCase()}</span> (오른쪽) 키로 더 "정상적인" 로고를 가능한 빠르게 고르세요.`,
    '반응시간이 중요합니다. 시간 제한이 있습니다(약 0.8초).',
  ],
  // 영상 파일 경로 (조건별)
  video: {
    aud0: 'videos/highlight_silent.mp4',   // (청각 방해 없음) - 관중/해설 최소화 버전
    aud1: 'videos/highlight_noise.mp4'     // (청각 방해 있음) - 응원/해설 믹스 버전
  },
  // 2AFC 시도 횟수(예: 48)
  n_trials: 48,
  // 자극 세트: 브랜드별 정답(진짜) 이미지와 오답(변형) 이미지 경로
  // familiar / unfamiliar는 분석 시 태깅용. 실험 화면은 이미지 경로만 사용.
  stimuli: [
    // 예시 — 경로를 본인 이미지로 교체하세요.
    {brand:'NIKE', fam:'familiar',  correct:'img/NIKE_true.png', foil:'img/NIKE_tilt2.png'},
    {brand:'ADIDAS', fam:'familiar',  correct:'img/ADIDAS_true.png', foil:'img/ADIDAS_spacing.png'},
    {brand:'PUMA', fam:'familiar',  correct:'img/PUMA_true.png', foil:'img/PUMA_round.png'},
    {brand:'LOTTE', fam:'unfamiliar', correct:'img/LOTTE_true.png', foil:'img/LOTTE_semibold.png'},
    {brand:'HANKOOK', fam:'unfamiliar', correct:'img/HANKOOK_true.png', foil:'img/HANKOOK_kerning.png'},
    {brand:'MOBIS', fam:'unfamiliar', correct:'img/MOBIS_true.png', foil:'img/MOBIS_slight.png'},
    // ... 브랜드를 12~16개 정도로 늘린 뒤, 블록 반복으로 총 n_trials 구성 권장
  ],
  // 데이터 저장: 파일로 저장(로컬 다운로드) 또는 Apps Script 엔드포인트로 전송(선택)
  save_mode: 'download', // 'download' | 'post'
  post_url: 'https://script.google.com/macros/s/your-deployment-id/exec' // 선택
};

// =============================
// 2) URL 파라미터: pid, cond(aud0|aud1)
// =============================
const urlParams = new URLSearchParams(window.location.search);
const PID = urlParams.get('pid') || `P${Math.random().toString(36).slice(2,8)}`;
const COND = (urlParams.get('cond') === 'aud1') ? 'aud1' : 'aud0';

// =============================
// 3) 시퀀스 유틸리티
// =============================
function shuffle(array){
  const a = array.slice();
  for(let i=a.length-1;i>0;i--){
    const j = Math.floor(Math.random()*(i+1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

// 좌/우 위치 랜덤 배치 생성
function makeTrials(base, n_total){
  // base(자극목록)를 반복·샘플링하여 n_total에 맞춤
  let pool = [];
  while(pool.length < n_total){ pool = pool.concat(shuffle(base)); }
  pool = pool.slice(0, n_total);
  // 각 트라이얼에서 정답이 왼/오 확률 50%
  return pool.map(stim => {
    const left_is_correct = Math.random() < 0.5;
    return {
      brand: stim.brand,
      fam: stim.fam,
      left:  left_is_correct ? stim.correct : stim.foil,
      right: left_is_correct ? stim.foil    : stim.correct,
      correct_key: left_is_correct ? EXP.left_key : EXP.right_key,
      correct_img: left_is_correct ? 'left' : 'right',
      correct_path: left_is_correct ? stim.correct : stim.correct // 그대로 기록용
    };
  });
}

const TRIALS = makeTrials(EXP.stimuli, EXP.n_trials);

// =============================
// 4) 프리로드(이미지/비디오)
// =============================
const preload = {
  type: jsPsychPreload,
  images: Array.from(new Set(TRIALS.flatMap(t => [t.left, t.right]))),
  video: [EXP.video.aud0, EXP.video.aud1]
};

// =============================
// 5) 타이틀 & PID 수집(선택)
// =============================
const welcome = {
  type: jsPsychInstructions,
  pages: [
    `<h2>스포츠 스폰서 로고 구분 과제</h2>
     <p>이 실험은 경기 하이라이트를 본 뒤, 두 개의 로고 중 더 정상적인 로고를 빠르게 고르는 과제입니다.</p>
     <p class="small">참여코드: <b>${PID}</b> / 조건: <b>${COND}</b></p>`
  ],
  show_clickable_nav: true,
  button_label_next: '다음'
};

const instr = {
  type: jsPsychInstructions,
  pages: EXP.instructions.map(txt => `<p>${txt}</p>`),
  show_clickable_nav: true,
  button_label_next: '다음'
};

// =============================
// 6) 비디오 블록(청각 방해 조작)
// =============================
const video_block = {
  type: jsPsychVideoButtonResponse,
  stimulus: [ EXP.video[COND] ],
  prompt: '<p class="center">다음 영상을 집중해서 시청하세요. 종료 후 "계속" 버튼이 나타납니다.</p>',
  choices: ['계속'],
  width: 800,
  response_ends_trial: false,
};

// =============================
// 7) 2AFC RT 과제 (fixation -> 자극 -> RT)
// =============================
const fixation = {
  type: jsPsychHtmlKeyboardResponse,
  stimulus: '<div class="center" style="font-size:48px;">+</div>',
  choices: 'NO_KEYS',
  trial_duration: EXP.fixation_ms,
};

const twoafc = {
  timeline: [
    fixation,
    {
      type: jsPsychHtmlKeyboardResponse,
      stimulus: () => {
        const t = jsPsych.timelineVariable('t');
        return `<div class="stimrow">
                  <img src="${t.left}" alt="left" />
                  <img src="${t.right}" alt="right" />
                </div>
                <p class="center small">왼쪽: <span class="kbd">${EXP.left_key.toUpperCase()}</span> / 오른쪽: <span class="kbd">${EXP.right_key.toUpperCase()}</span></p>`;
      },
      choices: [EXP.left_key, EXP.right_key],
      trial_duration: EXP.rt_deadline,
      data: () => {
        const t = jsPsych.timelineVariable('t');
        return {
          pid: PID,
          cond: COND,
          task: '2AFC',
          brand: t.brand,
          fam: t.fam,
          left: t.left,
          right: t.right,
          correct_key: t.correct_key
        };
      },
      on_finish: (data) => {
        data.correct = (data.response === data.correct_key) ? 1 : 0;
        data.timeout = (data.rt === null) ? 1 : 0;
      }
    },
    {
      type: jsPsychHtmlKeyboardResponse,
      stimulus: '<div class="center small">다음...</div>',
      trial_duration: EXP.isi_ms,
      choices: 'NO_KEYS'
    }
  ],
  timeline_variables: TRIALS.map(t => ({t})),
  randomize_order: true
};

// =============================
// 8) 조작 체크(선택)
// =============================
const noise_check = {
  type: jsPsychSurveyLikert,
  preamble: '<h3>조작 체크</h3>',
  questions: [
    {prompt:'관중 소리/해설 때문에 방해를 느꼈다', labels:['전혀 아님','', '', '', '', '', '매우 그럼'], required:true},
    {prompt:'스폰서를 의식적으로 보려 했다', labels:['전혀 아님','', '', '', '', '', '매우 그럼'], required:true}
  ],
  randomize_question_order: false
};

// =============================
// 9) 저장 & 종료
// =============================
function saveData(){
  const csv = jsPsych.data.get().csv();
  if(EXP.save_mode === 'post'){
    fetch(EXP.post_url, {method:'POST', mode:'no-cors', body: csv});
  } else {
    const blob = new Blob([csv], {type:'text/csv'});
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    const ts = new Date().toISOString().replace(/[:.]/g,'-');
    a.download = `rt_2afc_${PID}_${COND}_${ts}.csv`;
    a.click();
  }
}

const debrief = {
  type: jsPsychHtmlButtonResponse,
  stimulus: `<h3>참여해 주셔서 감사합니다.</h3>
             <p>데이터가 자동 저장(다운로드)됩니다. 파일을 안전한 곳에 보관해 주세요.</p>`,
  choices: ['종료'],
  on_finish: saveData
};

// =============================
// 10) jsPsych 초기화 & 타임라인 실행
// =============================
const jsPsych = initJsPsych({
  display_element: 'jspsych-target',
  on_finish: () => { /* 페이지 유지 */ }
});

const timeline = [preload, welcome, instr, video_block, twoafc, noise_check, debrief];
jsPsych.run(timeline);
</script>
</body>
</html>
