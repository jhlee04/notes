
✅ 포함된 기능

1. 검토 status, LLM zero-shot, LLM status3 → 통일된 그룹 (on-track, at-risk, 제외: completed)


2. 정확도 비교 (accuracy, f1-score 포함)


3. 시각화:

✅ 정확도 막대그래프

✅ confusion matrix 히트맵 (LLM status3 기준)





---

✅ 1. 상태 정규화 함수 (표현 다양성 대응)
```
def normalize_status(value):
    if not isinstance(value, str):
        return None
    val = value.strip().lower().replace(' ', '').replace('_', '').replace('-', '')
    
    if val in ['ontrack', 'proposal']:
        return 'on-track'
    elif val in ['atrisk', 'delayed']:
        return 'at-risk'
    elif val in ['completed', 'done', 'finished']:
        return 'completed'
    return None

```
---

✅ 2. 정규화 컬럼 생성
```
# 상태 정규화
merged_df['정답'] = merged_df['검토 status'].apply(normalize_status)
merged_df['zero-shot'] = merged_df['LLM zero-shot status'].apply(normalize_status)
merged_df['status3'] = merged_df['LLM_status3'].apply(normalize_status)

# 평가 대상: completed 제외
eval_df = merged_df[merged_df['정답'].isin(['on-track', 'at-risk'])]

```
---

✅ 3. 정확도 및 F1-score 계산
```
from sklearn.metrics import accuracy_score, f1_score

acc_zero = accuracy_score(eval_df['정답'], eval_df['zero-shot'])
acc_s3 = accuracy_score(eval_df['정답'], eval_df['status3'])

f1_zero = f1_score(eval_df['정답'], eval_df['zero-shot'], pos_label='at-risk')
f1_s3 = f1_score(eval_df['정답'], eval_df['status3'], pos_label='at-risk')

```
---

✅ 4. 정확도 & F1-score 막대그래프
```
import matplotlib.pyplot as plt

labels = ['Zero-shot', 'LLM status3']
accs = [acc_zero, acc_s3]
f1s = [f1_zero, f1_s3]

x = range(len(labels))
width = 0.35

plt.figure(figsize=(8, 5))
plt.bar(x, accs, width=width, label='Accuracy')
plt.bar([i + width for i in x], f1s, width=width, label='F1-score')
plt.xticks([i + width / 2 for i in x], labels)
plt.ylim(0, 1)
plt.ylabel("Score")
plt.title("LLM 상태 예측 정확도 비교")
plt.legend()
plt.grid(axis='y', linestyle='--', alpha=0.5)
plt.tight_layout()
plt.show()
```

---

✅ 5. Confusion Matrix (status3 기준)
```
from sklearn.metrics import confusion_matrix
import seaborn as sns

cm = confusion_matrix(eval_df['정답'], eval_df['status3'], labels=['on-track', 'at-risk'])

plt.figure(figsize=(5, 4))
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
            xticklabels=['on-track', 'at-risk'],
            yticklabels=['on-track', 'at-risk'])
plt.title("LLM status3 Confusion Matrix")
plt.xlabel("Predicted")
plt.ylabel("Actual")
plt.tight_layout()
plt.show()

```
---

✅ 출력 예시 요약

비교 항목	설명

Accuracy	전체 분류 정확도
F1-score	at-risk에 대한 민감도 중심 정밀도
Bar Plot	Zero-shot vs status3 성능 시각적 비교
Confusion Matrix	어떤 상태에서 오분류가 발생했는지 해석 가능



---

필요하시면 위 코드를 .py로 묶거나, Streamlit/PDF 보고서 자동화 버전도 제작해드릴 수 있습니다.
지금 이대로 실행해 보시겠어요?

