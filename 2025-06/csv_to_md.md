 

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

 
