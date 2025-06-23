 

✅ 1. CSV → Markdown 변환 함수 (필터 지원 포함)

import pandas as pd

def convert_filtered_csv_to_md(
    csv_path,
    output_md_path,
    keyword_filter=None,
    week_filter=None
):
    df = pd.read_csv(csv_path)

    # 필요한 컬럼만 추출
    columns_needed = ['주차', '담당자', '파트', '제목', '내용']
    df = df[[col for col in columns_needed if col in df.columns]]

    # 필터: 키워드 포함 여부
    if keyword_filter:
        keyword_filter = keyword_filter.lower()
        df = df[df['내용'].astype(str).str.lower().str.contains(keyword_filter)]

    # 필터: 주차 (단일 or 리스트 모두 처리)
    if week_filter:
        if isinstance(week_filter, list):
            df = df[df['주차'].isin(week_filter)]
        else:
            df = df[df['주차'] == week_filter]

    # Markdown 구성
    md_lines = []
    for _, row in df.iterrows():
        week = row.get('주차', '')
        owner = row.get('담당자', '')
        part = row.get('파트', '')
        title = row.get('제목', '')
        content = row.get('내용', '')
        md_lines.append(f"### [{week}주차] {part} - {title}")
        md_lines.append(f"- 담당자: {owner}")
        md_lines.append(f"- 내용: {content}\n")

    # 파일 저장
    with open(output_md_path, "w", encoding="utf-8") as f:
        f.write("\n".join(md_lines))

    return output_md_path


---

✅ 2. 프롬프트 생성 함수 (.md 파일 두 개 → 하나의 프롬프트 텍스트)

def generate_prompt(presentation_md_path, weekly_md_path):
    with open(presentation_md_path, "r", encoding="utf-8") as f:
        presentation_md = f.read()

    with open(weekly_md_path, "r", encoding="utf-8") as f:
        weekly_md = f.read()

    prompt_template = """
당신은 반도체 부서의 회의록 작성 전문가입니다.  
아래 발표자료 요약과 팀별 주간보고를 참고하여 회의록 초안을 작성하세요.

[발표자료 요약]
{presentation_md}

[주간보고 요약]
{weekly_md}

요청 형식:
- 회의 일시  
- 주제  
- 발표 요지  
- 논의 내용  
- 액션 아이템 (담당자, 마감일 포함)
""".strip()

    return prompt_template.format(
        presentation_md=presentation_md,
        weekly_md=weekly_md
    )


---

✅ 사용 예시
# Step 1: CSV → Markdown 변환
convert_filtered_csv_to_md(
    csv_path="data/weekly_report.csv",
    output_md_path="data/filtered_weekly_report.md",
    keyword_filter="yield",                  # 선택사항
    week_filter=[24, 25, 26]                 # 다중 주차 지원
)

# Step 2: 발표자료와 주간보고 md로 프롬프트 생성
prompt = generate_prompt(
    presentation_md_path="data/presentation_summary.md",
    weekly_md_path="data/filtered_weekly_report.md"
)

print(prompt)  # 또는 LLM 모델에 직접 전달




✅ 가능 여부: 발표자료.md + 주간보고.md → 회의록 초안

충분히 가능합니다.

이 구조는 다음과 같이 매핑됩니다:

자료 유형	내용	회의록 내 활용

발표자료.md	발표자가 준비한 핵심 안건 정리	회의 배경, 주요 의제 요약
주간보고.md	각 팀의 현황 및 이슈	논의 근거, 담당자 의견 맥락
(LLM 활용)	두 입력을 통합 → 회의 흐름 재구성	논의 내용, 결정사항, 액션아이템 작성



---

✅ 자체 LLM 활용이 더 유리한 이유 (Msty 기반 pipeline 구성)

🌟 장점

1. 반복 가능: 매주 같은 구조로 자동 처리 가능


2. 보안성: 외부 API 없이 폐쇄망 내 처리


3. 팀 커스터마이징: 조직 내 보고 스타일 반영한 프롬프트 튜닝 가능




---

🔧 추천 pipeline 구조 (코드 기반)

1. 📁 입력 정리

/input/presentation.md: 발표 자료 요약

/input/weekly_report.csv: 팀별 주간보고


2. 🧾 csv → md 요약 (자동 변환)

# 예시 변환 결과
## [팀] DRAM설계팀
- 과제: Cell Capacitor Reliability 개선
- 진척: 공정 실험 완료, 수율 개선 3%
- 이슈: 내습 공정 조건 불안정

3. 🧠 프롬프트 구성

template = """
당신은 반도체 부서의 회의록 작성 전문가입니다.  
아래 발표자료 요약과 팀별 주간보고를 참고하여 회의록 초안을 작성하세요.

[발표자료 요약]
{presentation_md}

[주간보고 요약]
{weekly_md}

요청 형식:
- 회의 일시  
- 주제  
- 발표 요지  
- 논의 내용  
- 액션 아이템 (담당자, 마감일 포함)
"""


---

⚙️ Msty LLM 호출 구조 (예시)

from msty import LLM  # 예시 이름, 실제 사용 환경 맞춰 수정

llm = LLM(model='your-local-model-name')  # llama.cpp, bge 기반 등

prompt = template.format(
    presentation_md=load_text("input/presentation.md"),
    weekly_md=convert_csv_to_md("input/weekly_report.csv")
)

result = llm.generate(prompt)
save_text("output/회의록_초안.md", result)


---
 
