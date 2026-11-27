# task-brief 주니어 화면 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 주니어가 목표를 쓰고, 확인 질문에 답하고, 완성된 프롬프트를 복사하는 화면을 정적 HTML/CSS/JS로 만든다.

**Architecture:** 페이지 한 장(`site/index.html`) 안에서 쓰기, 확인, 프롬프트 세 구역을 한 번에 하나만 보여 준다. 프롬프트 조립과 응답 정리는 DOM을 모르는 순수 함수로 분리해 Node에서 단위 테스트한다. 서버 호출은 `analyze()` 함수 하나 뒤에 두고, 서버 언어가 0단계에서 확정되기 전까지는 고정 응답을 돌려주는 대역으로 연결한다.

**Tech Stack:** HTML, CSS, 브라우저 ES 모듈(빌드 없음, 프레임워크 없음), Node 20 이상의 내장 테스트 러너(`node --test`). 외부 의존성 없음.

**Spec:** `docs/superpowers/specs/2026-11-27-task-brief-design.md`

**스펙과 다르게 잡은 한 가지:** 스펙은 AI 호출을 분석과 프롬프트 완성 두 번으로 적었다. 이 계획은 프롬프트를 브라우저에서 규칙대로 조립하므로 AI 호출은 분석 한 번이다. 답마다 프롬프트의 어느 구역에 어떤 줄로 들어갈지를 분석 응답이 함께 준다. 고정 부분을 AI가 쓰지 않는다는 스펙 요구를 그대로 지키면서 토큰이 줄어든다.

## Global Constraints

- 프레임워크, 빌드 도구, 외부 라이브러리를 쓰지 않는다. `package.json`에 dependencies를 두지 않는다.
- 스크립트가 하는 일은 서버 호출, 질문 표시, 프롬프트 조립, 복사뿐이다.
- 색은 `#FFFFFF`, `#111111`, `#6B6B6B`, `#E5E5E5`만 쓴다. 강조색은 없다. 오류 문구도 `#111111`에 굵게 쓴다.
- 글꼴은 Noto Sans KR 한 가지, 프롬프트만 고정폭 글꼴.
- 글자 크기는 28px, 16px, 14px 세 단계. 저작권 문구와 기본 규칙 안내만 13px.
- 입력 칸에 상자를 두지 않는다. 구역은 1px 선으로만 나눈다. 장식 요소를 두지 않는다.
- 화면마다 검은 버튼은 하나. 본문 폭은 최대 640px.
- 모든 화면 아래에 `© 2026 cttd.co.kr`.
- 확인 질문 최대 3개, 놓친 것 짚기 최대 3개.
- 필수 입력은 목표 한 칸.
- 모든 입력에 `label`을 연결한다. 키보드만으로 끝까지 진행할 수 있어야 한다. 누를 수 있는 요소의 높이는 44px 이상.
- 화면 문구에 "고르다/골라", "집약"을 쓰지 않는다. 위치나 상황 뜻의 "자리"를 쓰지 않는다.
- 주니어 화면에 관리자 화면 링크를 두지 않는다.
- 화면 문구는 해요체로 쓴다. 단정하지 않고 "~할 수 있어요"처럼 쓴다. 프롬프트 본문은 Claude Code가 읽는 글이므로 "~한다" 평서체를 유지한다.
- 완료 기준은 주니어가 처음부터 쓰지 않는다. 도구가 먼저 제안하고 주니어가 고치거나 그대로 둔다.
- 꼭 할 것과 하지 말 것은 프리셋을 체크로 정하고, 직접 작성도 받는다.
- 꼭 할 것, 하지 말 것, 자료는 처음부터 펼쳐 둔다. 눌러서 펼치는 버튼을 두지 않는다.
- 완성된 프롬프트는 화면에서 바로 고칠 수 있고, 복사는 고친 내용을 그대로 가져간다.

## 파일 구조

| 파일 | 책임 |
| --- | --- |
| `package.json` | 모듈 형식과 `test`, `dev` 스크립트 |
| `site/index.html` | 세 구역의 마크업 |
| `site/styles.css` | 전체 스타일 |
| `site/js/prompt-template.js` | 지시문 틀(프롬프트의 고정 부분)과 버전 번호 |
| `site/js/build-prompt.js` | 태스크 객체를 프롬프트 글로 조립 |
| `site/js/analysis.js` | 분석 응답 정리(상한 적용), 답을 태스크에 반영 |
| `site/js/api.js` | `analyze()` — 지금은 고정 응답 대역 |
| `site/js/app.js` | DOM 연결과 화면 전환 |
| `tests/build-prompt.test.js` | 프롬프트 조립 테스트 |
| `tests/analysis.test.js` | 응답 정리와 답 반영 테스트 |
| `scripts/dev-server.mjs` | 로컬 확인용 정적 서버 |

### 공용 자료 형태

모든 태스크가 아래 두 형태를 쓴다.

```js
// 태스크
{
  id: 'A12',            // 없으면 ''
  goal: '…',            // 필수
  done: '…',            // 없으면 ''
  must: ['…'],          // 없으면 []
  mustnot: ['…'],
  materials: ['…'],     // 피그마 주소나 파일 경로, 한 줄에 하나
  notes: ['…'],         // 확인 단계에서 정해진 내용
  skills: ['ui-component-builder'],
}

// 분석 응답 (정리된 뒤)
{
  taskId: 'A12',
  skills: ['ui-component-builder'],
  items: [
    // 서버가 done(제안하는 완료 기준)을 주면 정리 단계에서 맨 앞에 아래 항목이 생긴다.
    // { kind: 'done', section: 'done', text: '', why: '', options: [], draft: '제안 글' }
    {
      kind: 'question' | 'check',
      section: 'done' | 'must' | 'mustnot' | 'note',  // 답이 들어갈 구역
      text: '질문',
      why: '왜 묻는지 한 줄',                          // 없으면 ''
      options: [{ label: '보기', line: '프롬프트에 들어갈 줄' }],  // line이 ''이면 아무것도 넣지 않음
    },
  ],
}

// 답 (items와 같은 순서의 배열)
{ choice: 0 | 'custom' | 'none' | null, text: '직접 쓴 글' }
```

---

### Task 1: 지시문 틀과 프롬프트 조립

**Files:**
- Create: `package.json`
- Create: `site/js/prompt-template.js`
- Create: `site/js/build-prompt.js`
- Test: `tests/build-prompt.test.js`

**Interfaces:**
- Consumes: 없음
- Produces: `PROMPT_VERSION: string`, `RULES: string[]`, `SKILL_INSTALL: string`, `PSD_NOTE: string`, `WRAP_UP: string` (`prompt-template.js`), `buildPrompt(task): string` (`build-prompt.js`)

- [ ] **Step 1: `package.json` 작성**

```json
{
  "name": "task-brief",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "node --test",
    "dev": "node scripts/dev-server.mjs"
  }
}
```

- [ ] **Step 2: 실패하는 테스트 작성**

`tests/build-prompt.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { buildPrompt } from '../site/js/build-prompt.js';
import { RULES, WRAP_UP, SKILL_INSTALL, PSD_NOTE } from '../site/js/prompt-template.js';

const base = {
  id: '', goal: '이벤트 목록에 필터 추가', done: '',
  must: [], mustnot: [], materials: [], notes: [], skills: [],
};

test('목표만 있어도 기본 규칙과 마무리가 들어간다', () => {
  const out = buildPrompt(base);
  assert.equal(
    out,
    `목표\n이벤트 목록에 필터 추가\n\n기본 규칙\n- ${RULES[0]}\n- ${RULES[1]}\n\n마무리\n${WRAP_UP}`,
  );
});

test('빈 칸은 제목 줄도 남기지 않는다', () => {
  const outLines = buildPrompt(base).split('\n');
  for (const title of ['완료 기준', '꼭 할 것', '하지 말 것', '확인한 내용', '자료', '스킬']) {
    assert.ok(!outLines.includes(title), `${title} 구역이 남아 있다`);
  }
  assert.ok(!outLines.some((line) => line.startsWith('태스크 ')), '태스크 번호 줄이 남아 있다');
});

test('모든 칸이 차면 정해진 순서로 들어간다', () => {
  const out = buildPrompt({
    id: 'A12', goal: '목표글', done: '기준글',
    must: ['할것1', '할것2'], mustnot: ['말것1'],
    materials: ['https://figma.com/x'], notes: ['확인1'],
    skills: ['design-to-code-analysis', 'ui-component-builder'],
  });
  const order = ['태스크 A12', '목표\n목표글', '완료 기준\n기준글', '꼭 할 것\n- 할것1\n- 할것2',
    '하지 말 것\n- 말것1', '확인한 내용\n- 확인1', '자료\n- https://figma.com/x', '기본 규칙',
    `스킬\ndesign-to-code-analysis, ui-component-builder\n${SKILL_INSTALL}`, '마무리'];
  let from = 0;
  for (const part of order) {
    const at = out.indexOf(part, from);
    assert.ok(at >= from, `순서가 어긋났다: ${part}`);
    from = at + part.length;
  }
});

test('PSD 자료가 있으면 읽는 방법 안내가 붙는다', () => {
  const withPsd = buildPrompt({ ...base, materials: ['design/main.PSD'] });
  assert.ok(withPsd.includes(`- design/main.PSD\n${PSD_NOTE}`));
  const without = buildPrompt({ ...base, materials: ['design/main.png'] });
  assert.ok(!without.includes(PSD_NOTE));
});
```

- [ ] **Step 3: 테스트가 실패하는지 확인**

Run: `npm test`
Expected: FAIL — `Cannot find module '…/site/js/build-prompt.js'`

- [ ] **Step 4: 지시문 틀 작성**

`site/js/prompt-template.js`:

```js
// 프롬프트의 고정 부분. 문구를 고치면 PROMPT_VERSION을 올린다.
export const PROMPT_VERSION = '0.1.0';

export const RULES = [
  '작업 전에 프로젝트 규칙 문서(CLAUDE.md, README, 린트·포매터 설정)와 비슷한 기존 화면·컴포넌트를 먼저 읽고 그 방식을 따른다. 이미 있는 컴포넌트는 새로 만들지 않는다. 다르게 해야 하면 이유를 말하고 먼저 묻는다.',
  '요청한 것만 만든다. 요청하지 않은 기능, 옵션, 라이브러리를 넣지 않는다. HTML과 CSS로 되는 것에 스크립트를 쓰지 않는다.',
];

export const SKILL_INSTALL =
  '위 스킬이 설치돼 있지 않으면 그 스킬만 개인 영역에 내려받은 뒤 진행한다(npx skills add https://github.com/cttd/skills --skill <이름>). 프로젝트 폴더에는 설치하지 않는다. 설치에 실패하면 이유를 알리고 스킬 없이 진행한다.';

export const PSD_NOTE =
  'PSD는 연결된 도구가 있으면 그것으로 읽고, 없으면 직접 읽어 보고, 안 되면 PNG로 내보내 달라고 요청한다.';

export const WRAP_UP =
  '작업이 끝나면 완료 기준별 충족 여부, 기존 방식과 달라진 부분, 요청 범위를 넘은 부분을 보고한다.';
```

- [ ] **Step 5: 프롬프트 조립 작성**

`site/js/build-prompt.js`:

```js
import { RULES, SKILL_INSTALL, PSD_NOTE, WRAP_UP } from './prompt-template.js';

function block(title, body) {
  return body === '' ? '' : `${title}\n${body}`;
}

function list(lines) {
  return lines.map((line) => `- ${line}`).join('\n');
}

export function buildPrompt(task) {
  const hasPsd = task.materials.some((m) => /\.psd$/i.test(m));
  const materials = task.materials.length
    ? list(task.materials) + (hasPsd ? `\n${PSD_NOTE}` : '')
    : '';
  const skills = task.skills.length ? `${task.skills.join(', ')}\n${SKILL_INSTALL}` : '';

  return [
    task.id ? `태스크 ${task.id}` : '',
    block('목표', task.goal),
    block('완료 기준', task.done),
    block('꼭 할 것', list(task.must)),
    block('하지 말 것', list(task.mustnot)),
    block('확인한 내용', list(task.notes)),
    block('자료', materials),
    block('기본 규칙', list(RULES)),
    block('스킬', skills),
    block('마무리', WRAP_UP),
  ].filter(Boolean).join('\n\n');
}
```

- [ ] **Step 6: 테스트가 통과하는지 확인**

Run: `npm test`
Expected: PASS — 4 tests

- [ ] **Step 7: 커밋**

```bash
git add package.json site/js/prompt-template.js site/js/build-prompt.js tests/build-prompt.test.js
git commit -m "feat: 지시문 틀과 프롬프트 조립 추가"
```

---

### Task 2: 분석 응답 정리와 답 반영

**Files:**
- Create: `site/js/analysis.js`
- Test: `tests/analysis.test.js`

**Interfaces:**
- Consumes: 없음 (공용 자료 형태만 따른다)
- Produces: `normalizeAnalysis(raw): { taskId: string, skills: string[], items: Item[] }`, `applyAnswers(task, items, answers): task`

- [ ] **Step 1: 실패하는 테스트 작성**

`tests/analysis.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { normalizeAnalysis, applyAnswers } from '../site/js/analysis.js';

const item = (kind, n) => ({ kind, section: 'must', text: `${kind}${n}`, options: [{ label: '네', line: `줄${n}` }] });

test('질문과 짚기는 각각 3개까지만 남는다', () => {
  const raw = {
    taskId: 'A1', skills: ['visual-qa'],
    items: [1, 2, 3, 4, 5].flatMap((n) => [item('question', n), item('check', n)]),
  };
  const out = normalizeAnalysis(raw);
  assert.equal(out.items.filter((i) => i.kind === 'question').length, 3);
  assert.equal(out.items.filter((i) => i.kind === 'check').length, 3);
  assert.deepEqual(out.items.map((i) => i.text),
    ['question1', 'question2', 'question3', 'check1', 'check2', 'check3']);
});

test('형식이 깨진 응답은 질문 없이 진행하게 빈 결과가 된다', () => {
  for (const raw of [null, undefined, 'x', 42, {}, { items: 'no' }]) {
    assert.deepEqual(normalizeAnalysis(raw), { taskId: '', skills: [], items: [] });
  }
});

test('글이 없는 항목과 보기는 버리고, 모르는 구역은 note로 바꾼다', () => {
  const out = normalizeAnalysis({
    items: [
      { kind: 'check', section: 'must', text: '   ' },
      null,
      { kind: 'weird', section: 'nowhere', text: ' 질문 ', why: ' 이유 ',
        options: [{ label: ' 네 ', line: ' 줄 ' }, { label: '' }, null, { label: '아니요' }] },
    ],
  });
  assert.deepEqual(out.items, [{
    kind: 'question', section: 'note', text: '질문', why: '이유',
    options: [{ label: '네', line: '줄' }, { label: '아니요', line: '' }],
  }]);
});

test('제안된 완료 기준은 맨 앞 항목이 되고 상한에 세지 않는다', () => {
  const out = normalizeAnalysis({ done: ' 필터가 유지된다 ', items: [1, 2, 3].map((n) => item('question', n)) });
  assert.equal(out.items.length, 4);
  assert.deepEqual(out.items[0],
    { kind: 'done', section: 'done', text: '', why: '', options: [], draft: '필터가 유지된다' });
});

const task = { id: 'A1', goal: 'g', done: '', must: ['기존'], mustnot: [], materials: [], notes: [], skills: [] };
const items = [
  { kind: 'question', section: 'done', text: 'q', why: '', options: [{ label: 'a', line: '기준줄' }] },
  { kind: 'check', section: 'must', text: 'c1', why: '', options: [{ label: 'a', line: '할것줄' }, { label: 'b', line: '' }] },
  { kind: 'check', section: 'mustnot', text: 'c2', why: '', options: [] },
  { kind: 'check', section: 'note', text: 'c3', why: '', options: [{ label: 'a', line: '확인줄' }] },
];

test('보기를 정하면 그 줄이 구역에 들어간다', () => {
  const out = applyAnswers(task, items, [
    { choice: 0, text: '' }, { choice: 0, text: '' }, { choice: 'custom', text: ' 직접쓴줄 ' }, { choice: 0, text: '' },
  ]);
  assert.equal(out.done, '기준줄');
  assert.deepEqual(out.must, ['기존', '할것줄']);
  assert.deepEqual(out.mustnot, ['직접쓴줄']);
  assert.deepEqual(out.notes, ['확인줄']);
});

test('해당 없음, 답 없음, 줄이 빈 보기는 아무것도 넣지 않는다', () => {
  const out = applyAnswers(task, items, [
    { choice: null, text: '' }, { choice: 1, text: '' }, { choice: 'none', text: '쓰다 만 글' }, { choice: 'custom', text: '  ' },
  ]);
  assert.equal(out.done, '');
  assert.deepEqual(out.must, ['기존']);
  assert.deepEqual(out.mustnot, []);
  assert.deepEqual(out.notes, []);
});

test('원래 태스크는 바뀌지 않는다', () => {
  applyAnswers(task, items, [{ choice: 0, text: '' }, { choice: 0, text: '' }]);
  assert.equal(task.done, '');
  assert.deepEqual(task.must, ['기존']);
});
```

- [ ] **Step 2: 테스트가 실패하는지 확인**

Run: `npm test`
Expected: FAIL — `Cannot find module '…/site/js/analysis.js'`

- [ ] **Step 3: 구현 작성**

`site/js/analysis.js`:

```js
const SECTIONS = new Set(['done', 'must', 'mustnot', 'note']);
const MAX_PER_KIND = 3;

function cleanOption(raw) {
  if (!raw || typeof raw.label !== 'string' || raw.label.trim() === '') return null;
  return { label: raw.label.trim(), line: typeof raw.line === 'string' ? raw.line.trim() : '' };
}

function cleanItem(raw) {
  if (!raw || typeof raw.text !== 'string' || raw.text.trim() === '') return null;
  return {
    kind: raw.kind === 'check' ? 'check' : 'question',
    section: SECTIONS.has(raw.section) ? raw.section : 'note',
    text: raw.text.trim(),
    why: typeof raw.why === 'string' ? raw.why.trim() : '',
    options: Array.isArray(raw.options) ? raw.options.map(cleanOption).filter(Boolean) : [],
  };
}

export function normalizeAnalysis(raw) {
  const src = raw && typeof raw === 'object' ? raw : {};
  const all = (Array.isArray(src.items) ? src.items : []).map(cleanItem).filter(Boolean);
  const done = typeof src.done === 'string' ? src.done.trim() : '';
  const doneItem = { kind: 'done', section: 'done', text: '', why: '', options: [], draft: done };
  return {
    taskId: typeof src.taskId === 'string' ? src.taskId : '',
    skills: Array.isArray(src.skills) ? src.skills.filter((s) => typeof s === 'string' && s !== '') : [],
    items: [
      ...(done === '' ? [] : [doneItem]),
      ...all.filter((i) => i.kind === 'question').slice(0, MAX_PER_KIND),
      ...all.filter((i) => i.kind === 'check').slice(0, MAX_PER_KIND),
    ],
  };
}

function lineFor(item, answer) {
  if (!answer) return '';
  if (answer.choice === 'custom') return answer.text.trim();
  if (typeof answer.choice === 'number' && item.options[answer.choice]) return item.options[answer.choice].line;
  return '';
}

export function applyAnswers(task, items, answers) {
  const out = { ...task, must: [...task.must], mustnot: [...task.mustnot], notes: [...task.notes] };
  items.forEach((item, i) => {
    const line = lineFor(item, answers[i]);
    if (line === '') return;
    if (item.section === 'done') out.done = line;
    else if (item.section === 'must') out.must.push(line);
    else if (item.section === 'mustnot') out.mustnot.push(line);
    else out.notes.push(line);
  });
  return out;
}
```

- [ ] **Step 4: 테스트가 통과하는지 확인**

Run: `npm test`
Expected: PASS — 11 tests (Task 1의 4개 포함)

- [ ] **Step 5: 커밋**

```bash
git add site/js/analysis.js tests/analysis.test.js
git commit -m "feat: 분석 응답 정리와 답 반영 추가"
```

---

### Task 3: 화면 마크업과 스타일

**Files:**
- Create: `site/index.html`
- Create: `site/styles.css`
- Create: `scripts/dev-server.mjs`

**Interfaces:**
- Consumes: 없음
- Produces: Task 4가 쓰는 요소 id — `step-write`, `goal`, `goal-error`, `must`, `mustnot`, `materials`, `write-next`, `write-status`, `step-check`, `goal-echo`, `check-progress`, `check-text`, `check-why`, `check-draft-wrap`, `check-draft`, `check-options`, `check-custom-wrap`, `check-custom`, `check-prev`, `step-prompt`, `prompt-id`, `prompt-title`, `prompt-notice`, `prompt-text`, `copy`, `copy-status`, `restart`.  프리셋 체크박스의 `name`은 `must-preset`, `mustnot-preset`이고 `value`가 프롬프트에 들어갈 줄이다.

- [ ] **Step 1: 로컬 확인용 서버 작성**

ES 모듈은 `file://`로 열면 막히므로 확인할 때 이 서버를 쓴다.

`scripts/dev-server.mjs`:

```js
import { createServer } from 'node:http';
import { readFile } from 'node:fs/promises';
import { extname, join, normalize } from 'node:path';
import { fileURLToPath } from 'node:url';

const root = join(fileURLToPath(new URL('.', import.meta.url)), '..', 'site');
const types = { '.html': 'text/html; charset=utf-8', '.css': 'text/css; charset=utf-8', '.js': 'text/javascript; charset=utf-8' };

createServer(async (req, res) => {
  const path = normalize(decodeURIComponent(new URL(req.url, 'http://localhost').pathname));
  const file = join(root, path === '/' || path === '\\' ? 'index.html' : path);
  if (!file.startsWith(root)) { res.writeHead(403).end(); return; }
  try {
    const body = await readFile(file);
    res.writeHead(200, { 'content-type': types[extname(file)] ?? 'application/octet-stream' }).end(body);
  } catch {
    res.writeHead(404).end('not found');
  }
}).listen(5173, () => console.log('http://localhost:5173'));
```

- [ ] **Step 2: 마크업 작성**

`site/index.html`:

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>task-brief</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;500;700&display=swap">
<link rel="stylesheet" href="styles.css">
</head>
<body>
<div class="page">
<header class="top"><span class="name">task-brief</span></header>

<main class="main">

<form id="step-write" class="step" novalidate>
  <div class="goal">
    <label for="goal" class="label">목표</label>
    <textarea id="goal" class="big" rows="2" placeholder="무엇을, 왜 만드나요?" aria-describedby="goal-error"></textarea>
    <p id="goal-error" class="error" hidden>목표를 써 주세요.</p>
  </div>

  <div class="rows">
    <fieldset class="group">
      <legend class="label">꼭 할 것</legend>
      <label class="check"><input type="checkbox" name="must-preset" value="좁은 화면(모바일 너비)에서도 레이아웃이 깨지지 않을 것">좁은 화면에서도 깨지지 않게</label>
      <label class="check"><input type="checkbox" name="must-preset" value="키보드만으로 모든 기능을 쓸 수 있을 것">키보드만으로 쓸 수 있게</label>
      <label class="check"><input type="checkbox" name="must-preset" value="내용이 없을 때, 글이 길 때, 오류일 때의 화면도 만들 것">내용이 없을 때, 글이 길 때, 오류일 때 화면도 만들기</label>
      <label class="check"><input type="checkbox" name="must-preset" value="디자인 시안과 간격, 글자 크기, 색이 같을 것">디자인 시안과 똑같이</label>
      <div class="own">
        <label for="must" class="label">직접 작성</label>
        <textarea id="must" rows="1" placeholder="한 줄에 하나씩"></textarea>
      </div>
    </fieldset>
    <fieldset class="group">
      <legend class="label">하지 말 것</legend>
      <label class="check"><input type="checkbox" name="mustnot-preset" value="공통 CSS와 공통 컴포넌트를 수정하지 않는다">공통 CSS와 공통 컴포넌트 건드리지 않기</label>
      <label class="check"><input type="checkbox" name="mustnot-preset" value="작업 범위 밖의 기존 마크업 구조를 바꾸지 않는다">기존 마크업 구조 바꾸지 않기</label>
      <label class="check"><input type="checkbox" name="mustnot-preset" value="기존 파일 이름과 폴더 구조를 바꾸지 않는다">파일 이름과 폴더 구조 바꾸지 않기</label>
      <label class="check"><input type="checkbox" name="mustnot-preset" value="다른 페이지에 영향을 주는 변경을 하지 않는다">다른 페이지에 영향 주지 않기</label>
      <div class="own">
        <label for="mustnot" class="label">직접 작성</label>
        <textarea id="mustnot" rows="1" placeholder="한 줄에 하나씩"></textarea>
      </div>
    </fieldset>
    <div class="row">
      <label for="materials" class="label">자료</label>
      <textarea id="materials" rows="1" placeholder="피그마 주소나 파일 경로. 한 줄에 하나씩"></textarea>
    </div>
  </div>

  <div class="actions">
    <p class="note">기존 프로젝트 방식을 따른다, 요청한 것만 만든다.<br>이 두 가지는 항상 들어가요.</p>
    <button type="submit" class="primary" id="write-next">다음</button>
  </div>
  <p id="write-status" class="status" role="status"></p>
</form>

<form id="step-check" class="step" hidden novalidate>
  <p id="goal-echo" class="echo"></p>
  <fieldset class="question">
    <legend>
      <span id="check-progress" class="label"></span>
      <span id="check-text" class="big" tabindex="-1"></span>
    </legend>
    <p id="check-why" class="why"></p>
    <div id="check-draft-wrap" class="draft" hidden>
      <textarea id="check-draft" rows="3" aria-labelledby="check-text"></textarea>
    </div>
    <div id="check-options" class="options"></div>
    <div id="check-custom-wrap" class="custom" hidden>
      <label for="check-custom" class="label">직접 작성</label>
      <input id="check-custom" type="text">
    </div>
  </fieldset>
  <div class="actions">
    <button type="button" class="text" id="check-prev">이전</button>
    <button type="submit" class="primary">다음</button>
  </div>
</form>

<section id="step-prompt" class="step" hidden>
  <div class="actions end">
    <div class="head">
      <span id="prompt-id" class="label"></span>
      <h1 id="prompt-title" class="big" tabindex="-1">Claude Code에 붙여 넣으세요.</h1>
    </div>
    <button type="button" class="primary" id="copy">복사</button>
  </div>
  <p id="prompt-notice" class="status"></p>
  <div class="goal">
    <label for="prompt-text" class="label">프롬프트 · 바로 고칠 수 있어요</label>
    <textarea id="prompt-text" class="prompt" rows="20" spellcheck="false"></textarea>
  </div>
  <p id="copy-status" class="status" role="status"></p>
  <button type="button" class="text" id="restart">새 태스크</button>
</section>

</main>

<footer class="foot">© 2026 cttd.co.kr</footer>
</div>
<script type="module" src="js/app.js"></script>
</body>
</html>
```

프리셋 문구는 처음 넣는 초안이다. 시니어가 검토해 이 파일에서 바로 고친다.

- [ ] **Step 3: 스타일 작성**

`site/styles.css`:

```css
:root {
  --ink: #111111;
  --gray: #6B6B6B;
  --line: #E5E5E5;
  --paper: #FFFFFF;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--paper);
  color: var(--ink);
  font-family: 'Noto Sans KR', system-ui, sans-serif;
  font-size: 16px;
  line-height: 1.7;
}

[hidden] { display: none !important; }

.page {
  min-height: 100vh;
  padding: 28px 40px 48px;
  display: flex;
  flex-direction: column;
}

.top .name { font-size: 15px; font-weight: 700; }

.main {
  width: 100%;
  max-width: 640px;
  margin: 0 auto;
  padding-top: 96px;
}

.step { display: flex; flex-direction: column; gap: 40px; }

.foot { margin-top: auto; padding-top: 64px; font-size: 13px; color: var(--gray); }

.label { font-size: 14px; color: var(--gray); }

.big {
  margin: 0;
  font-size: 28px;
  font-weight: 500;
  line-height: 1.45;
  letter-spacing: -0.01em;
}

textarea, input[type="text"] {
  width: 100%;
  padding: 0;
  border: 0;
  background: transparent;
  color: inherit;
  font: inherit;
  resize: none;
  field-sizing: content;
}

textarea::placeholder, input::placeholder { color: var(--gray); }

textarea:focus, input[type="text"]:focus { outline: none; }

button:focus-visible,
input[type="radio"]:focus-visible,
input[type="checkbox"]:focus-visible,
[tabindex="-1"]:focus-visible {
  outline: 2px solid var(--ink);
  outline-offset: 3px;
}

[tabindex="-1"]:focus { outline: none; }

.goal { display: flex; flex-direction: column; gap: 12px; }

.error { margin: 0; font-size: 14px; font-weight: 700; }

.rows { display: flex; flex-direction: column; border-bottom: 1px solid var(--line); }

.row {
  display: flex;
  flex-wrap: wrap;
  align-items: baseline;
  gap: 4px 24px;
  padding: 14px 0;
  border-top: 1px solid var(--line);
}

.row .label { flex: 0 0 64px; }
.row textarea { flex: 1 1 240px; min-width: 0; min-height: 28px; }

.group {
  margin: 0;
  padding: 18px 0 14px;
  border: 0;
  border-top: 1px solid var(--line);
  min-width: 0;
}

.group legend { float: left; width: 100%; padding: 0 0 6px; }

.check {
  clear: both;
  display: flex;
  align-items: center;
  gap: 14px;
  min-height: 44px;
  cursor: pointer;
}

.check input { margin: 0; accent-color: var(--ink); }
.check:has(input:checked) { font-weight: 700; }

.own {
  display: flex;
  flex-wrap: wrap;
  align-items: baseline;
  gap: 4px 24px;
  padding-top: 10px;
}

.own .label { flex: 0 0 64px; }
.own textarea { flex: 1 1 240px; min-width: 0; min-height: 28px; }

.row:focus-within, .group:focus-within { border-top-color: var(--ink); }

button { font: inherit; color: inherit; cursor: pointer; }

.text {
  min-height: 44px;
  padding: 0;
  border: 0;
  background: transparent;
  font-size: 15px;
  color: var(--gray);
  align-self: flex-start;
}

.text:hover { color: var(--ink); }

.primary {
  min-height: 48px;
  padding: 0 28px;
  border: 0;
  border-radius: 6px;
  background: var(--ink);
  color: var(--paper);
  font-size: 15px;
  font-weight: 500;
}

.primary:disabled { background: var(--gray); cursor: default; }

.actions {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: center;
  gap: 16px 24px;
}

.actions.end { align-items: flex-end; }

.head { flex: 1 1 280px; display: flex; flex-direction: column; gap: 14px; }

.note { margin: 0; flex: 1 1 280px; font-size: 13px; color: var(--gray); }

.status { margin: 0; font-size: 14px; color: var(--gray); }
.status:empty { display: none; }

.echo { margin: 0; font-size: 14px; color: var(--gray); }

.question { margin: 0; padding: 0; border: 0; min-width: 0; }

.question legend {
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.why { margin: 14px 0 0; font-size: 15px; color: var(--gray); }
.why:empty { display: none; }

.draft { margin-top: 28px; }

.draft textarea {
  padding: 18px 0;
  border-top: 1px solid var(--ink);
  border-bottom: 1px solid var(--ink);
}

.options { margin-top: 28px; display: flex; flex-direction: column; border-bottom: 1px solid var(--line); }

.option {
  display: flex;
  align-items: center;
  gap: 14px;
  min-height: 56px;
  border-top: 1px solid var(--line);
  cursor: pointer;
}

.option input { margin: 0; accent-color: var(--ink); }
.option:has(input:checked) { font-weight: 700; }

.custom {
  margin-top: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding-bottom: 14px;
  border-bottom: 1px solid var(--ink);
}

textarea.prompt {
  padding: 28px 0;
  border-top: 1px solid var(--line);
  border-bottom: 1px solid var(--line);
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 14px;
  line-height: 1.75;
  overflow-wrap: anywhere;
}

textarea.prompt:focus { border-color: var(--ink); }

@media (max-width: 600px) {
  .page { padding: 20px 20px 40px; }
  .main { padding-top: 48px; }
  .big { font-size: 22px; }
}
```

- [ ] **Step 4: 브라우저에서 확인**

Run: `npm run dev` 후 `http://localhost:5173` 열기. 이 시점에는 `js/app.js`가 없어 콘솔에 404가 한 줄 나오는 것이 정상이다.

확인 목록:
- 화면에 "task-brief", "목표" 이름표와 큰 입력 칸, "꼭 할 것"과 "하지 말 것"의 체크 네 개씩과 "직접 작성" 칸, "자료" 칸, 기본 규칙 두 줄, 검은 "다음" 버튼, 맨 아래 "© 2026 cttd.co.kr"이 보인다. 눌러서 펼치는 버튼은 없다.
- 확인 구역과 프롬프트 구역은 보이지 않는다.
- 입력 칸에 상자가 없고, 색은 검정, 회색, 흰색뿐이다.
- 창 너비를 375px로 줄여도 가로 스크롤이 생기지 않는다.
- `#step-check`와 `#step-prompt`의 `hidden`을 지우면 각 구역이 시안과 같은 구성으로 보인다.

- [ ] **Step 5: 커밋**

```bash
git add site/index.html site/styles.css scripts/dev-server.mjs
git commit -m "feat: 주니어 화면 마크업과 스타일 추가"
```

---

### Task 4: 화면 연결

**Files:**
- Create: `site/js/api.js`
- Create: `site/js/app.js`

**Interfaces:**
- Consumes: `buildPrompt(task)` (Task 1), `normalizeAnalysis(raw)`, `applyAnswers(task, items, answers)` (Task 2), Task 3의 요소 id와 프리셋 `name`
- Produces: `analyze(task): Promise<object>` (`api.js`) — 서버가 붙을 때 이 함수의 안쪽만 바꾼다

- [ ] **Step 1: 서버 대역 작성**

`site/js/api.js`:

```js
// 서버가 붙기 전까지 쓰는 대역이다. 서버가 생기면 이 함수 안쪽만
// fetch 호출로 바꾼다. 돌려주는 형태는 그대로 둔다.
// done과 line은 프롬프트에 들어가므로 평서체, text와 why와 label은 화면 문구이므로 해요체.
export async function analyze(task) {
  return {
    taskId: `T${Date.now().toString(36).toUpperCase()}`,
    skills: ['ui-component-builder'],
    done: '브라우저에서 직접 확인했을 때 목표에 적은 대로 동작한다.',
    items: [
      {
        kind: 'check', section: 'must',
        text: '내용이 하나도 없을 때 화면은 어떻게 보이나요?',
        why: '빈 상태 처리가 없으면 화면이 비어 보여서 오류로 오해받을 수 있어요.',
        options: [{ label: '안내 문구를 보여 줘요', line: '내용이 없으면 안내 문구를 보여 줄 것' }],
      },
      {
        kind: 'check', section: 'note',
        text: '비슷한 화면이나 컴포넌트가 이미 있나요?',
        why: '있으면 새로 만들지 않고 그대로 써야 화면끼리 방식이 같아져요.',
        options: [
          { label: '있어요', line: '비슷한 기존 화면이 있다. 먼저 찾아서 그 방식을 따른다' },
          { label: '없어요', line: '' },
          { label: '모르겠어요', line: '비슷한 기존 화면이 있는지 먼저 찾아본다' },
        ],
      },
    ],
  };
}
```

- [ ] **Step 2: 화면 연결 작성**

`site/js/app.js`:

```js
import { analyze } from './api.js';
import { normalizeAnalysis, applyAnswers } from './analysis.js';
import { buildPrompt } from './build-prompt.js';

const $ = (id) => document.getElementById(id);
const steps = { write: $('step-write'), check: $('step-check'), prompt: $('step-prompt') };

let state = { task: null, items: [], answers: [], index: 0, notice: '' };

function show(name, focusTarget) {
  for (const [key, el] of Object.entries(steps)) el.hidden = key !== name;
  focusTarget.focus();
}

function lines(id) {
  return $(id).value.split('\n').map((s) => s.trim()).filter(Boolean);
}

function presets(name) {
  return [...document.querySelectorAll(`input[name="${name}"]:checked`)].map((el) => el.value);
}

function readTask() {
  return {
    id: '',
    goal: $('goal').value.trim(),
    done: '',
    must: [...presets('must-preset'), ...lines('must')],
    mustnot: [...presets('mustnot-preset'), ...lines('mustnot')],
    materials: lines('materials'),
    notes: [],
    skills: [],
  };
}

// 쓰기
steps.write.addEventListener('submit', async (event) => {
  event.preventDefault();
  const task = readTask();
  const missing = task.goal === '';
  $('goal-error').hidden = !missing;
  $('goal').setAttribute('aria-invalid', String(missing));
  if (missing) { $('goal').focus(); return; }

  $('write-next').disabled = true;
  $('write-status').textContent = '읽고 있어요.';
  let analysis;
  let notice = '';
  try {
    analysis = normalizeAnalysis(await analyze(task));
  } catch {
    analysis = normalizeAnalysis(null);
    notice = '확인 질문을 불러오지 못해서 입력한 내용만으로 만들었어요.';
  }
  $('write-next').disabled = false;
  $('write-status').textContent = '';

  task.id = analysis.taskId;
  task.skills = analysis.skills;
  state = {
    task,
    items: analysis.items,
    answers: analysis.items.map(() => ({ choice: null, text: '' })),
    index: 0,
    notice,
  };
  if (state.items.length === 0) showPrompt();
  else showItem();
});

// 확인
function addOption(value, label, checked) {
  const wrap = document.createElement('label');
  wrap.className = 'option';
  const radio = document.createElement('input');
  radio.type = 'radio';
  radio.name = 'choice';
  radio.value = value;
  radio.checked = checked;
  wrap.append(radio, label);
  $('check-options').append(wrap);
}

function showItem() {
  const item = state.items[state.index];
  const answer = state.answers[state.index];
  const isDone = item.kind === 'done';

  $('goal-echo').textContent = state.task.goal;
  $('check-progress').textContent = `${state.index + 1} / ${state.items.length}`;
  $('check-text').textContent = isDone ? '완료 기준을 이렇게 잡아 봤어요.' : item.text;
  $('check-why').textContent = isDone ? '맞으면 그대로 두고, 다르면 고쳐 주세요.' : item.why;

  $('check-draft-wrap').hidden = !isDone;
  $('check-options').hidden = isDone;
  $('check-options').replaceChildren();
  $('check-custom-wrap').hidden = true;
  $('check-custom').value = '';

  if (isDone) {
    $('check-draft').value = answer.choice === 'custom' ? answer.text : item.draft;
  } else {
    item.options.forEach((option, i) => addOption(String(i), option.label, answer.choice === i));
    addOption('custom', '직접 작성', answer.choice === 'custom');
    if (item.kind === 'check') addOption('none', '해당 없음', answer.choice === 'none');
    $('check-custom').value = answer.text;
    $('check-custom-wrap').hidden = answer.choice !== 'custom';
  }
  show('check', $('check-text'));
}

$('check-options').addEventListener('change', (event) => {
  const custom = event.target.value === 'custom';
  $('check-custom-wrap').hidden = !custom;
  if (custom) $('check-custom').focus();
});

function saveAnswer() {
  if (state.items[state.index].kind === 'done') {
    state.answers[state.index] = { choice: 'custom', text: $('check-draft').value.trim() };
    return;
  }
  const checked = steps.check.querySelector('input[name="choice"]:checked');
  let choice = null;
  if (checked) choice = checked.value === 'custom' || checked.value === 'none' ? checked.value : Number(checked.value);
  state.answers[state.index] = { choice, text: $('check-custom').value.trim() };
}

steps.check.addEventListener('submit', (event) => {
  event.preventDefault();
  saveAnswer();
  if (state.index + 1 < state.items.length) { state.index += 1; showItem(); }
  else showPrompt();
});

$('check-prev').addEventListener('click', () => {
  saveAnswer();
  if (state.index > 0) { state.index -= 1; showItem(); }
  else show('write', $('goal'));
});

// 프롬프트
function showPrompt() {
  const task = applyAnswers(state.task, state.items, state.answers);
  $('prompt-id').textContent = task.id;
  $('prompt-notice').textContent = state.notice;
  $('prompt-text').value = buildPrompt(task);
  $('copy-status').textContent = '';
  show('prompt', $('prompt-title'));
}

$('copy').addEventListener('click', async () => {
  try {
    await navigator.clipboard.writeText($('prompt-text').value);
    $('copy-status').textContent = '복사했어요.';
  } catch {
    $('prompt-text').select();
    $('copy-status').textContent = '자동 복사가 막혀서 전체를 선택해 뒀어요. Ctrl+C로 복사하세요.';
  }
});

$('restart').addEventListener('click', () => {
  location.reload();
});
```

`새 태스크`는 페이지를 다시 불러온다. 체크와 입력 내용이 모두 처음 상태로 돌아가므로 되돌리는 코드를 따로 두지 않는다.

완료 기준 화면에서 글을 모두 지우고 넘어가면 프롬프트에 완료 기준 구역이 들어가지 않는다.

- [ ] **Step 3: 단위 테스트가 그대로 통과하는지 확인**

Run: `npm test`
Expected: PASS — 11 tests

- [ ] **Step 4: 브라우저에서 전체 흐름 확인**

Run: `npm run dev` 후 `http://localhost:5173` 열기. 콘솔에 오류가 없어야 한다.

마우스 없이 키보드만으로 아래를 차례로 한다.

1. 목표를 비운 채 `Tab`으로 "다음"까지 가서 `Enter` → "목표를 써 주세요."가 나오고 포커스가 목표 칸으로 돌아간다.
2. 목표에 "이벤트 목록에 필터 추가"를 쓴다. `Tab`으로 "꼭 할 것"의 첫 체크로 가서 `Space`로 "좁은 화면에서도 깨지지 않게"를 체크하고, 그 아래 "직접 작성"에 "필터 상태가 주소에 남을 것"을 쓴다.
3. "다음"에서 `Enter` → 확인 화면으로 바뀌고, 위에 목표가 회색으로 보이고, "1 / 3"과 "완료 기준을 이렇게 잡아 봤어요."가 나온다. 아래 입력 칸에 제안 글이 들어 있다. 포커스는 제목 글에 있다.
4. `Tab`으로 입력 칸에 들어가 끝에 " 새로 고쳐도 필터가 유지된다."를 덧붙인다. "다음"으로 가서 `Enter`.
5. "2 / 3"에서 "안내 문구를 보여 줘요"를 정하고 `Enter`. 이유 한 줄이 질문 아래에 회색으로 보이는지 확인한다.
6. "3 / 3"에서 "이전"을 두 번 눌러 완료 기준 화면까지 돌아가면 덧붙인 글이 남아 있다. "다음"을 두 번 눌러 "3 / 3"으로 온다.
7. "3 / 3"에서 "해당 없음"을 정하고 `Enter` → 프롬프트 화면. 포커스는 "Claude Code에 붙여 넣으세요."에 있다.
8. 프롬프트에 아래가 순서대로 들어 있다: `태스크 T…`, 목표, `완료 기준` 아래 제안 글과 덧붙인 글, `꼭 할 것` 아래 세 줄("좁은 화면(모바일 너비)에서도 레이아웃이 깨지지 않을 것", "필터 상태가 주소에 남을 것", "내용이 없으면 안내 문구를 보여 줄 것"), 기본 규칙 두 줄, `스킬` 아래 "ui-component-builder"와 설치 안내, 마무리. `하지 말 것`, `확인한 내용`, `자료` 구역은 없다.
9. 프롬프트 칸에 들어가 맨 끝에 "테스트"를 덧붙인다. "복사"에서 `Enter` → "복사했어요."가 나온다. 메모장에 붙여 넣으면 덧붙인 "테스트"까지 들어 있다.
10. "새 태스크"에서 `Enter` → 빈 첫 화면으로 돌아가고 체크가 모두 풀려 있다.

서버 실패 처리도 확인한다. `site/js/api.js`의 `analyze` 첫 줄에 `throw new Error('test');`를 임시로 넣고 새로 고친 뒤 목표만 쓰고 "다음" → 확인 화면 없이 프롬프트 화면으로 가고 "확인 질문을 불러오지 못해서 입력한 내용만으로 만들었어요."가 보인다. 확인 뒤 임시 줄을 지운다.

- [ ] **Step 5: 커밋**

```bash
git add site/js/api.js site/js/app.js
git commit -m "feat: 쓰기, 확인, 프롬프트 화면 연결"
```

---

## 이 계획에 넣지 않은 것

| 항목 | 다룰 곳 |
| --- | --- |
| 서버 스크립트와 실제 AI 호출 (완료 기준 제안, 질문 생성) | 0단계에서 서버 언어 확정 뒤 별도 계획 |
| 태스크에 맞춘 프리셋 (지금은 고정 목록) | 서버 계획 |
| 파일 올리기 (지금은 주소나 경로를 글로 적음) | 서버 계획 |
| 기록 저장, 관리자 화면 | 2단계 계획 |
| Claude Code 사용량 수집 | 0단계 시험 뒤 2단계 계획 |
| 팀 공용 암호 | 서버 계획 |
| FTP 배포 | 서버 계획 |
