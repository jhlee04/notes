 
## ✅ \[가정된 입력 데이터 구조]

### 📄 `df_실적`

\| 팀 | 과제 | 최종목표 | 월 | 목표 | 실적 | 코멘트 |

### 📄 `df_회의`

\| 팀 | 과제 | 일자 | 내용 | status |

---

## ✅ \[전체 코드] 프롬프트 생성 + 사내 LLM 실행

```python
import pandas as pd
import requests
import time

# ✅ 사내 LLM API 설정 (예시용 URL, 실제 환경에 맞게 수정하세요)
LLM_API_URL = "http://internal-llm.company.com/api/chat"

def call_llm(prompt, model="qwen-turbo"):
    payload = {
        "model": model,
        "messages": [
            {"role": "system", "content": "당신은 기술 과제를 진단하는 리뷰 전문가입니다."},
            {"role": "user", "content": prompt}
        ],
        "temperature": 0.3,
        "max_tokens": 500
    }
    try:
        response = requests.post(LLM_API_URL, json=payload)
        return response.json()['choices'][0]['message']['content']
    except Exception as e:
        return f"[에러]: {str(e)}"

# ✅ 고유 과제 리스트 추출
과제목록 = df_실적[['팀', '과제', '최종목표']].drop_duplicates()
결과리스트 = []

for _, row in 과제목록.iterrows():
    팀, 과제, 최종목표 = row['팀'], row['과제'], row['최종목표']

    # 📌 실적 필터링 및 마크다운 구성
    실적_df = df_실적[(df_실적['팀'] == 팀) & (df_실적['과제'] == 과제)]
    실적_md = "\n".join([
        f"- {r['월']}: 목표={r['목표']}, 실적={r['실적']}, 코멘트={r['코멘트']}"
        for _, r in 실적_df.iterrows()
    ])

    # 📌 회의록 필터링 및 마크다운 구성
    회의_df = df_회의[(df_회의['팀'] == 팀) & (df_회의['과제'] == 과제)]
    회의_md = "\n".join([
        f"- [{r['일자']}] 내용: {r['내용']} / 상태: {r['status']}"
        for _, r in 회의_df.iterrows()
    ])

    # ✅ 프롬프트 마크다운 생성
    프롬프트 = f"""
# ✅ 과제 리뷰 요청

**팀명:** {팀}  
**과제명:** {과제}  
**최종 목표:** {최종목표}

---

## 📊 월별 실적 요약
{실적_md}

---

## 📝 회의 요약 (총 {len(회의_df)}회)
{회의_md}

---

## 🧠 질문
1. 이 과제는 최종 목표 달성을 향해 잘 진행되고 있나요? (순항 / 리스크 / 목표 변경 필요)  
2. 판단의 근거를 요약해 주세요.  
3. 향후 관리 포인트를 1문장으로 제안해 주세요.
"""

    # ✅ LLM 호출
    응답 = call_llm(프롬프트)

    결과리스트.append({
        "팀": 팀,
        "과제": 과제,
        "진단결과": 응답
    })

    time.sleep(1.2)  # API 과부하 방지

# ✅ 결과 저장
df_결과 = pd.DataFrame(결과리스트)
df_결과.to_csv("과제진단결과.csv", index=False)
```

---

## ✅ 출력 결과 예시 (`df_결과`)

| 팀    | 과제        | 진단결과                                  |
| ---- | --------- | ------------------------------------- |
| DRAM | Capacitor | 1. 부분 리스크 … 2. 판단 근거 … 3. 향후 관리 포인트 … |

---
 
