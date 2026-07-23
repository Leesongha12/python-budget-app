# 콘솔 가계부 서비스

Python 표준 라이브러리만 사용하여 구현한 파일 기반 콘솔 가계부 프로그램입니다.

수입과 지출 거래를 영구 저장하고, 거래 추가·조회·검색·수정·삭제, 월별 요약, 예산 관리, 카테고리 관리, CSV 가져오기·내보내기 기능을 제공합니다.

단순히 데이터를 저장하는 데서 끝나지 않고 다음과 같은 유지보수 가능한 구조를 목표로 구현합니다.

* 모델, 저장소, 서비스, CLI 계층 분리
* JSONL 기반 영구 저장
* 제너레이터를 이용한 거래 데이터 스트리밍
* 임시 파일과 원자적 교체를 이용한 안전한 수정·삭제
* 데코레이터를 이용한 예외 처리와 실행 로그 분리
* 타입 힌트를 이용한 함수 입출력 계약 명시
* 사용자 친화적인 오류 메시지와 종료 코드 제공

---

## 1. 개발 환경

* Python 3.10 이상
* 외부 라이브러리 미사용
* Python 표준 라이브러리만 사용

별도의 `pip install` 과정은 필요하지 않습니다.

Python 버전을 확인합니다.

```bash
python --version
```

또는 환경에 따라 다음 명령을 사용할 수 있습니다.

```bash
python3 --version
```

---

## 2. 프로젝트 구조

```text
project/
├── budget_app/
│   ├── __init__.py
│   ├── __main__.py
│   ├── cli.py
│   ├── models.py
│   ├── repositories.py
│   ├── services.py
│   ├── decorators.py
│   ├── validators.py
│   └── formatter.py
│
├── data/
│   ├── transactions.jsonl
│   ├── categories.jsonl
│   └── budgets.jsonl
│
├── exports/
├── logs/
│   └── budget_app.log
│
└── README.md
```

### 모듈별 책임

| 모듈                | 책임                                |
| ----------------- | --------------------------------- |
| `__main__.py`     | 프로그램 진입점                          |
| `cli.py`          | 명령어와 옵션 파싱, 대화형 입력                |
| `models.py`       | `Transaction`, `Budget` 등의 데이터 모델 |
| `repositories.py` | JSONL 파일 읽기·쓰기·수정·삭제              |
| `services.py`     | 거래, 예산, 카테고리 관련 비즈니스 로직           |
| `decorators.py`   | 예외 처리, 실행 로그, 실행 시간 측정            |
| `validators.py`   | 날짜, 금액, 거래 타입, 카테고리 검증            |
| `formatter.py`    | 콘솔 출력 형식과 테이블 정렬                  |

---

## 3. 실행 방법

프로젝트의 최상위 디렉터리에서 다음 형식으로 실행합니다.

```bash
python -m budget_app <command> [options]
```

전체 도움말을 확인합니다.

```bash
python -m budget_app --help
```

특정 명령어의 도움말도 확인할 수 있습니다.

```bash
python -m budget_app add --help
python -m budget_app search --help
python -m budget_app summary --help
```

---

## 4. 저장 폴더 변경

기본 데이터 저장 폴더는 프로젝트의 `./data`입니다.

다른 폴더를 사용하려면 전역 옵션인 `--data-dir`을 지정합니다.

```bash
python -m budget_app --data-dir ./my_data list
```

지정한 폴더가 존재하지 않으면 프로그램이 자동으로 생성합니다.

폴더 안에는 다음 세 파일이 생성됩니다.

```text
my_data/
├── transactions.jsonl
├── categories.jsonl
└── budgets.jsonl
```

---

## 5. 최초 실행

저장 폴더 또는 저장 파일이 존재하지 않으면 프로그램이 자동으로 생성합니다.

카테고리 파일이 비어 있으면 다음 기본 카테고리가 등록됩니다.

* `food`
* `transport`
* `rent`
* `salary`
* `shopping`
* `medical`
* `education`
* `entertainment`
* `utilities`
* `etc`

기본 카테고리는 `category add`와 `category remove` 명령으로 변경할 수 있습니다.

---

## 6. 주요 명령어

| 명령어        | 설명               |
| ---------- | ---------------- |
| `add`      | 거래 추가            |
| `list`     | 거래 목록 조회         |
| `search`   | 조건별 거래 검색        |
| `summary`  | 월별 수입·지출 요약      |
| `budget`   | 월별 예산 설정·조회      |
| `category` | 카테고리 추가·조회·삭제    |
| `update`   | 거래 수정            |
| `delete`   | 거래 삭제            |
| `import`   | CSV 파일에서 거래 가져오기 |
| `export`   | 거래를 CSV 파일로 내보내기 |

---

# 거래 관리

## 7. 거래 추가

`add` 명령은 대화형 입력 방식으로 실행됩니다.

```bash
python -m budget_app add
```

입력 예시:

```text
날짜(YYYY-MM-DD): 2024-01-15
타입(income/expense): expense
카테고리: food
금액(양수 정수): 15000
메모(선택): 점심
태그(쉼표로 구분, 없으면 엔터): meal,work
[저장 완료] id=TX-000012
```

### 거래 필드

| 필드         | 필수    | 설명                    |
| ---------- | ----- | --------------------- |
| `id`       | 자동 생성 | 거래를 구분하는 유일한 ID       |
| `date`     | 필수    | `YYYY-MM-DD` 형식       |
| `type`     | 필수    | `income` 또는 `expense` |
| `category` | 필수    | 등록된 카테고리              |
| `amount`   | 필수    | 0보다 큰 정수              |
| `memo`     | 선택    | 거래 설명                 |
| `tags`     | 선택    | 여러 태그 입력 가능           |

### 거래 ID

거래 ID는 다음과 같은 형식으로 자동 생성됩니다.

```text
TX-000001
TX-000002
TX-000003
```

삭제된 ID는 다시 사용하지 않습니다.

---

## 8. 거래 목록 조회

최근 거래를 최신순으로 출력합니다.

```bash
python -m budget_app list
```

기본 출력 건수는 20건입니다.

출력 건수를 제한하려면 `--limit` 옵션을 사용합니다.

```bash
python -m budget_app list --limit 3
```

출력 예시:

```text
TX-000012 | 2024-01-15 | expense | food      |      15,000원 | 점심
TX-000011 | 2024-01-14 | income  | salary    |   3,000,000원 |
TX-000010 | 2024-01-12 | expense | transport |      20,000원 |
```

거래 파일은 한 번에 전부 메모리에 올리지 않고 제너레이터를 통해 한 줄씩 읽습니다.

---

## 9. 거래 검색

`search` 명령으로 여러 조건을 조합하여 거래를 검색할 수 있습니다.

```bash
python -m budget_app search [options]
```

### 검색 옵션

| 옵션                  | 설명                    |
| ------------------- | --------------------- |
| `--from YYYY-MM-DD` | 검색 시작일                |
| `--to YYYY-MM-DD`   | 검색 종료일                |
| `--category NAME`   | 카테고리                  |
| `--type TYPE`       | `income` 또는 `expense` |
| `--q KEYWORD`       | 메모에 포함된 문자열           |
| `--tag TAG`         | 특정 태그                 |
| `--limit N`         | 최대 출력 건수              |

### 기간 검색

```bash
python -m budget_app search --from 2024-01-01 --to 2024-01-31
```

### 카테고리 검색

```bash
python -m budget_app search --category food
```

### 지출만 검색

```bash
python -m budget_app search --type expense
```

### 메모 키워드 검색

```bash
python -m budget_app search --q 점심
```

### 태그 검색

```bash
python -m budget_app search --tag work
```

### 조건 조합

```bash
python -m budget_app search \
  --from 2024-01-01 \
  --to 2024-01-31 \
  --category food \
  --type expense \
  --tag meal
```

검색 결과는 최신 거래부터 출력됩니다.

검색 조건에 맞는 거래가 없으면 다음과 같이 출력합니다.

```text
[안내] 검색 조건에 맞는 거래가 없습니다.
```

---

## 10. 거래 수정

거래 수정은 **옵션 기반 방식**으로 고정되어 있습니다.

```bash
python -m budget_app update --id <거래 ID> [수정 옵션]
```

수정하려는 필드만 옵션으로 전달합니다.

### 수정 가능한 옵션

| 옵션                      | 설명            |
| ----------------------- | ------------- |
| `--date YYYY-MM-DD`     | 날짜 수정         |
| `--type income/expense` | 거래 타입 수정      |
| `--category NAME`       | 카테고리 수정       |
| `--amount NUMBER`       | 금액 수정         |
| `--memo TEXT`           | 메모 수정         |
| `--tags TAGS`           | 쉼표로 구분된 태그 수정 |

### 금액 수정

```bash
python -m budget_app update --id TX-000012 --amount 18000
```

### 카테고리와 메모 수정

```bash
python -m budget_app update \
  --id TX-000012 \
  --category food \
  --memo "팀 점심 식사"
```

### 태그 수정

```bash
python -m budget_app update \
  --id TX-000012 \
  --tags "meal,work,team"
```

성공 예시:

```text
[수정 완료] id=TX-000012
```

존재하지 않는 ID를 입력한 경우:

```text
[오류] id=TX-999999에 해당하는 거래가 없습니다.
[힌트] list 또는 search 명령으로 거래 ID를 확인해 주세요.
```

수정할 필드를 하나도 입력하지 않으면 오류로 처리합니다.

```text
[오류] 수정할 항목이 지정되지 않았습니다.
[힌트] --amount, --memo 등의 수정 옵션을 입력해 주세요.
```

---

## 11. 거래 삭제

거래 ID를 기준으로 거래를 삭제합니다.

```bash
python -m budget_app delete --id TX-000012
```

성공 예시:

```text
[삭제 완료] id=TX-000012
```

존재하지 않는 ID를 입력한 경우:

```text
[오류] id=TX-999999에 해당하는 거래가 없습니다.
[힌트] list 또는 search 명령으로 거래 ID를 확인해 주세요.
```

거래 수정과 삭제는 기존 파일을 직접 덮어쓰지 않습니다.

다음 순서로 안전하게 처리합니다.

1. 원본 파일을 한 줄씩 읽습니다.
2. 변경 내용을 임시 파일에 기록합니다.
3. 기록이 정상적으로 끝났는지 확인합니다.
4. `os.replace()`를 사용하여 원본 파일과 임시 파일을 원자적으로 교체합니다.

작업 도중 오류가 발생하면 기존 파일이 유지됩니다.

---

# 월별 요약과 예산

## 12. 월별 요약

특정 월의 수입, 지출, 잔액을 출력합니다.

```bash
python -m budget_app summary --month 2024-01
```

기본 카테고리별 지출 순위는 상위 3개입니다.

```bash
python -m budget_app summary --month 2024-01 --top 5
```

출력 예시:

```text
2024-01 월별 요약
────────────────────────────────
총 수입: 3,000,000원
총 지출:   215,000원
잔액:     2,785,000원

지출 TOP 3
1. rent            150,000원
2. food             45,000원
3. transport        20,000원
```

해당 월에 거래가 없으면 다음과 같이 출력합니다.

```text
[안내] 2024-01에는 등록된 거래가 없습니다.
```

---

## 13. 예산 설정

월별 지출 예산을 설정합니다.

```bash
python -m budget_app budget set \
  --month 2024-01 \
  --amount 500000
```

출력 예시:

```text
[저장 완료] 2024-01 예산 500,000원
```

이미 예산이 설정된 월에 다시 실행하면 기존 예산을 변경합니다.

```text
[수정 완료] 2024-01 예산 600,000원
```

### 예산 조회

특정 월의 예산을 조회합니다.

```bash
python -m budget_app budget get --month 2024-01
```

출력 예시:

```text
2024-01 예산: 500,000원
```

전체 예산 목록을 조회합니다.

```bash
python -m budget_app budget list
```

---

## 14. 예산이 포함된 월별 요약

해당 월에 예산이 설정되어 있으면 `summary` 결과에 예산 사용률이 표시됩니다.

```bash
python -m budget_app summary --month 2024-01 --top 3
```

출력 예시:

```text
2024-01 월별 요약
────────────────────────────────
총 수입: 3,000,000원
총 지출:   215,000원
잔액:     2,785,000원

예산:       500,000원
사용률:          43.0%
남은 예산:  285,000원

지출 TOP 3
1. rent            150,000원
2. food             45,000원
3. transport        20,000원
```

예산을 초과한 경우 경고 메시지가 출력됩니다.

```text
예산:       500,000원
사용률:         112.0%
초과 금액:   60,000원

[경고] 이번 달 지출이 설정된 예산을 초과했습니다.
```

수입 거래는 예산 사용률 계산에 포함하지 않습니다.

---

# 카테고리 관리

## 15. 카테고리 목록

등록된 카테고리를 확인합니다.

```bash
python -m budget_app category list
```

출력 예시:

```text
등록된 카테고리
- education
- entertainment
- etc
- food
- medical
- rent
- salary
- shopping
- transport
- utilities
```

---

## 16. 카테고리 추가

카테고리 추가는 대화형으로 실행할 수 있습니다.

```bash
python -m budget_app category add
```

입력 예시:

```text
카테고리명: subscription
[저장 완료] category=subscription
```

옵션으로도 추가할 수 있습니다.

```bash
python -m budget_app category add --name subscription
```

이미 존재하는 카테고리를 추가하면 오류로 처리합니다.

```text
[오류] category=subscription은 이미 등록되어 있습니다.
[힌트] category list 명령으로 현재 카테고리를 확인해 주세요.
```

카테고리명은 앞뒤 공백을 제거하고 소문자로 저장합니다.

---

## 17. 카테고리 삭제

```bash
python -m budget_app category remove --name subscription
```

해당 카테고리를 사용 중인 거래가 없으면 삭제됩니다.

```text
[삭제 완료] category=subscription
```

거래에서 사용 중인 카테고리는 삭제할 수 없습니다.

```text
[오류] category=food를 사용하는 거래가 8건 존재합니다.
[힌트] 해당 거래의 카테고리를 먼저 수정한 후 다시 삭제해 주세요.
```

이 프로그램은 데이터 일관성을 위해 사용 중인 카테고리를 다른 카테고리로 자동 변경하지 않습니다.

---

# CSV 가져오기와 내보내기

## 18. CSV 스키마

가져오기와 내보내기는 UTF-8 인코딩과 헤더가 포함된 CSV 파일을 사용합니다.

| 열 이름       | 필수 | 형식 및 설명               |
| ---------- | -- | --------------------- |
| `date`     | Y  | `YYYY-MM-DD`          |
| `type`     | Y  | `income` 또는 `expense` |
| `category` | Y  | 등록된 카테고리              |
| `amount`   | Y  | 0보다 큰 정수              |
| `memo`     | N  | 문자열                   |
| `tags`     | N  | 쉼표로 구분된 문자열           |

CSV 예시:

```csv
date,type,category,amount,memo,tags
2024-01-15,expense,food,15000,점심,"meal,work"
2024-01-25,income,salary,3000000,1월 급여,"salary,monthly"
2024-01-27,expense,transport,20000,교통카드 충전,transport
```

CSV 파일에는 `id` 열을 작성하지 않습니다.

거래 ID는 가져오기 과정에서 프로그램이 새로 생성합니다.

---

## 19. CSV 가져오기

CSV 파일의 거래를 현재 거래 저장소에 추가합니다.

```bash
python -m budget_app import --from import.csv
```

출력 예시:

```text
[완료] imported=5, skipped=0
```

일부 행에 오류가 있는 경우 정상 행은 저장하고 오류 행은 건너뜁니다.

```text
[경고] 4번째 행을 건너뜁니다: 등록되지 않은 카테고리입니다.
[완료] imported=4, skipped=1
```

### 가져오기 검증 항목

* 필수 헤더 존재 여부
* 날짜 형식
* 거래 타입
* 금액이 양수 정수인지 여부
* 카테고리 등록 여부
* UTF-8 파일 여부

등록되지 않은 카테고리가 포함되어 있으면 해당 행을 건너뜁니다.

먼저 카테고리를 등록한 후 다시 가져와야 합니다.

```bash
python -m budget_app category add --name insurance
python -m budget_app import --from import.csv
```

---

## 20. CSV 내보내기

거래를 CSV 파일로 저장합니다.

내보내기에는 다음 조건 중 하나 이상을 반드시 지정해야 합니다.

* `--month YYYY-MM`
* `--from YYYY-MM-DD`와 `--to YYYY-MM-DD`

### 월 단위 내보내기

```bash
python -m budget_app export \
  --out exports/2024-01.csv \
  --month 2024-01
```

출력 예시:

```text
[완료] exports/2024-01.csv (12 records)
```

### 기간 단위 내보내기

```bash
python -m budget_app export \
  --out exports/first-quarter.csv \
  --from 2024-01-01 \
  --to 2024-03-31
```

### 추가 조건과 조합

```bash
python -m budget_app export \
  --out exports/food-expense.csv \
  --month 2024-01 \
  --category food \
  --type expense
```

조건을 지정하지 않으면 오류로 처리합니다.

```text
[오류] 내보내기 범위가 지정되지 않았습니다.
[힌트] --month 또는 --from과 --to 옵션을 입력해 주세요.
```

내보내기 결과가 0건이어도 헤더가 포함된 CSV 파일을 생성합니다.

```text
[완료] exports/empty.csv (0 records)
```

---

# 저장 파일

## 21. 거래 파일

경로:

```text
data/transactions.jsonl
```

한 줄에 하나의 JSON 객체가 저장됩니다.

```json
{"id":"TX-000012","date":"2024-01-15","type":"expense","category":"food","amount":15000,"memo":"점심","tags":["meal","work"]}
{"id":"TX-000011","date":"2024-01-14","type":"income","category":"salary","amount":3000000,"memo":"","tags":[]}
```

---

## 22. 카테고리 파일

경로:

```text
data/categories.jsonl
```

저장 예시:

```json
{"name":"food"}
{"name":"transport"}
{"name":"rent"}
{"name":"salary"}
```

---

## 23. 예산 파일

경로:

```text
data/budgets.jsonl
```

저장 예시:

```json
{"month":"2024-01","amount":500000}
{"month":"2024-02","amount":600000}
```

동일한 월에는 하나의 예산만 존재합니다.

기존 월의 예산을 다시 설정하면 해당 값이 변경됩니다.

---

# 설계 특징

## 24. 제너레이터 스트리밍

거래 조회와 검색은 거래 파일 전체를 리스트로 변환하지 않고 한 줄씩 처리합니다.

개념적인 구현 형태는 다음과 같습니다.

```python
from collections.abc import Iterator

def iter_transactions(self) -> Iterator[Transaction]:
    with self.path.open("r", encoding="utf-8") as file:
        for line in file:
            if line.strip():
                yield Transaction.from_json(line)
```

`yield`를 사용하면 거래 파일의 크기가 커져도 모든 데이터를 한 번에 메모리에 저장하지 않습니다.

이를 통해 다음과 같은 장점이 있습니다.

* 대용량 파일에서도 메모리 사용량이 급격히 증가하지 않음
* 필요한 데이터만 순차적으로 처리 가능
* 검색, 요약, 내보내기 로직에서 같은 스트림 재사용 가능
* 저장소와 비즈니스 로직의 결합 감소

최신순 출력이 필요한 경우 제한된 개수만 유지하는 방식이나 역방향 파일 읽기 구조를 사용하여 전체 거래 객체를 한 번에 적재하지 않도록 설계합니다.

---

## 25. 데코레이터

공통 관심사는 명령어 함수 내부에 반복해서 작성하지 않고 데코레이터로 분리합니다.

적용 대상의 예시는 다음과 같습니다.

* 예상 가능한 애플리케이션 오류 처리
* 사용자용 오류 메시지 출력
* 명령어 실행 로그 기록
* 명령어 실행 시간 측정
* 오류 종료 코드 변환

개념적인 사용 예시는 다음과 같습니다.

```python
@handle_cli_errors
@log_execution
def run_summary_command(args: argparse.Namespace) -> int:
    ...
```

이를 통해 명령어 함수는 실제 기능 수행에만 집중할 수 있습니다.

---

## 26. 타입 힌트

함수의 매개변수와 반환값에 타입 힌트를 적용합니다.

```python
def find_by_id(self, transaction_id: str) -> Transaction | None:
    ...
```

```python
def calculate_monthly_summary(
    self,
    month: str,
    top_n: int,
) -> MonthlySummary:
    ...
```

타입 힌트를 통해 다음 내용을 명확히 알 수 있습니다.

* 함수에 어떤 값을 전달해야 하는지
* 함수가 어떤 값을 반환하는지
* 값이 없을 가능성이 있는지
* 리스트, 딕셔너리, 제너레이터 내부에 어떤 타입이 들어가는지

타입 힌트는 실행 중 강제로 타입을 제한하지는 않지만, 코드 리뷰와 유지보수 과정에서 함수의 계약을 명확하게 해 줍니다.

---

## 27. 계층별 책임

### 모델 계층

데이터 자체의 구조와 직렬화 규칙을 담당합니다.

예시:

* `Transaction`
* `Budget`
* `MonthlySummary`

### 저장소 계층

파일 입출력과 데이터 보존을 담당합니다.

예시:

* 거래 스트리밍
* JSONL 추가 저장
* 거래 파일 재작성
* 임시 파일 생성
* 원자적 파일 교체

### 서비스 계층

업무 규칙을 담당합니다.

예시:

* 등록된 카테고리인지 확인
* 월별 수입과 지출 계산
* 예산 사용률 계산
* 사용 중인 카테고리 삭제 차단
* 검색 조건 적용

### CLI 계층

사용자 입력과 콘솔 출력을 담당합니다.

예시:

* `argparse` 명령어 구성
* 대화형 입력
* 옵션값 전달
* 결과 메시지 출력
* 종료 코드 반환

---

# 입력 검증과 오류 처리

## 28. 날짜 검증

날짜는 실제로 존재하는 `YYYY-MM-DD` 형식이어야 합니다.

잘못된 입력:

```text
2024-13-40
```

출력:

```text
[오류] 날짜 형식이 올바르지 않습니다.
[힌트] YYYY-MM-DD 형식으로 입력해 주세요. 예: 2024-01-15
```

---

## 29. 금액 검증

금액은 0보다 큰 정수여야 합니다.

잘못된 값:

```text
0
-1000
12.5
만원
```

출력:

```text
[오류] 금액은 0보다 큰 정수여야 합니다.
[힌트] 쉼표나 단위를 제외하고 숫자만 입력해 주세요. 예: 15000
```

---

## 30. 거래 타입 검증

거래 타입은 다음 두 값만 허용합니다.

```text
income
expense
```

잘못된 값이 입력된 경우:

```text
[오류] 허용되지 않은 거래 타입입니다.
[힌트] income 또는 expense 중 하나를 입력해 주세요.
```

---

## 31. 카테고리 검증

거래 추가, 수정, 가져오기에서 사용하는 카테고리는 카테고리 저장소에 등록되어 있어야 합니다.

```text
[오류] category=insurance가 등록되어 있지 않습니다.
[힌트] category add 명령으로 카테고리를 먼저 추가해 주세요.
```

---

## 32. 오류 메시지 정책

예상 가능한 사용자 입력 오류는 스택트레이스를 출력하지 않습니다.

오류 메시지는 다음 두 부분으로 구성합니다.

```text
[오류] 오류가 발생한 원인
[힌트] 사용자가 문제를 해결할 수 있는 방법
```

예상하지 못한 내부 오류가 발생한 경우에도 전체 스택트레이스를 사용자 화면에 출력하지 않습니다.

```text
[오류] 명령을 처리하는 중 예상하지 못한 문제가 발생했습니다.
[힌트] logs/budget_app.log 파일을 확인해 주세요.
```

개발 및 점검에 필요한 상세 정보는 로그 파일에 기록합니다.

---

## 33. 종료 코드

| 종료 코드 | 의미               |
| ----- | ---------------- |
| `0`   | 정상 종료            |
| `1`   | 일반 애플리케이션 오류     |
| `2`   | 명령어 또는 옵션 사용 오류  |
| `3`   | 입력값 검증 오류        |
| `4`   | 파일 읽기·쓰기 오류      |
| `5`   | 요청한 데이터가 존재하지 않음 |

터미널에서 직전 명령의 종료 코드를 확인할 수 있습니다.

Linux 또는 macOS:

```bash
echo $?
```

Windows PowerShell:

```powershell
$LASTEXITCODE
```

---

# 명령어 사용 예시

## 34. 기본 사용 흐름

카테고리를 확인합니다.

```bash
python -m budget_app category list
```

거래를 추가합니다.

```bash
python -m budget_app add
```

최근 거래를 확인합니다.

```bash
python -m budget_app list --limit 10
```

특정 월의 지출을 검색합니다.

```bash
python -m budget_app search \
  --from 2024-01-01 \
  --to 2024-01-31 \
  --type expense
```

월 예산을 설정합니다.

```bash
python -m budget_app budget set \
  --month 2024-01 \
  --amount 500000
```

월별 요약을 확인합니다.

```bash
python -m budget_app summary \
  --month 2024-01 \
  --top 3
```

월별 거래를 CSV로 내보냅니다.

```bash
python -m budget_app export \
  --out exports/2024-01.csv \
  --month 2024-01
```

---

## 35. 도움말 예시

전체 명령어:

```bash
python -m budget_app --help
```

거래 검색:

```bash
python -m budget_app search --help
```

거래 수정:

```bash
python -m budget_app update --help
```

CSV 내보내기:

```bash
python -m budget_app export --help
```

---

# 데이터 안전성

## 36. 데이터 저장 정책

거래 추가는 기존 JSONL 파일의 마지막에 새로운 거래를 추가하는 방식으로 처리합니다.

수정과 삭제처럼 기존 데이터 변경이 필요한 작업은 원본 파일을 직접 수정하지 않습니다.

```text
transactions.jsonl
        ↓ 읽기
transactions.tmp
        ↓ 검증 완료
os.replace()
        ↓
transactions.jsonl
```

임시 파일 작성이 완료되기 전에 오류가 발생하면 원본 파일은 그대로 유지됩니다.

---

## 37. 손상된 데이터 처리

JSONL 파일에서 일부 행이 올바른 JSON 형식이 아니면 해당 행을 무조건 무시하지 않습니다.

다음과 같이 오류 위치와 해결 방법을 출력합니다.

```text
[오류] transactions.jsonl의 12번째 줄을 읽을 수 없습니다.
[힌트] 파일의 JSON 형식을 확인하거나 백업 파일로 복구해 주세요.
```

읽기 오류가 발생한 상태에서는 데이터 손상을 방지하기 위해 수정·삭제 작업을 중단합니다.

---

# 선택 기능

## 38. 백업

백업 기능을 구현한 경우 다음과 같이 실행합니다.

```bash
python -m budget_app backup
```

백업 파일은 타임스탬프가 포함된 폴더에 생성됩니다.

```text
backups/
└── 20240131_153012/
    ├── transactions.jsonl
    ├── categories.jsonl
    └── budgets.jsonl
```

출력 예시:

```text
[백업 완료] backups/20240131_153012
```

---

# 기능 점검표

## 39. 필수 기능

* [x] 거래 추가
* [x] 거래 목록 조회
* [x] 조건별 거래 검색
* [x] 월별 수입·지출 요약
* [x] 월별 예산 설정과 조회
* [x] 예산 사용률과 초과 경고
* [x] 카테고리 추가·조회·삭제
* [x] 사용 중인 카테고리 삭제 차단
* [x] 거래 수정
* [x] 거래 삭제
* [x] CSV 가져오기
* [x] CSV 내보내기
* [x] 3개 이상의 파일로 데이터 영구 저장
* [x] 제너레이터 기반 거래 스트리밍
* [x] 데코레이터 적용
* [x] 타입 힌트 적용
* [x] 최소 3개 이상의 모듈 분리
* [x] 사용자용 오류 메시지
* [x] 오류 발생 시 0이 아닌 종료 코드
* [x] 모든 명령의 `--help` 지원

---

## 40. 핵심 학습 내용

이 프로젝트를 통해 다음 내용을 학습할 수 있습니다.

1. JSONL 파일을 이용한 데이터 영구 저장
2. 파일 기반 데이터의 생성·조회·수정·삭제
3. 제너레이터와 `yield`를 이용한 스트리밍
4. `dataclass`를 이용한 데이터 모델 정의
5. 저장소와 서비스 계층의 책임 분리
6. 데코레이터를 이용한 공통 관심사 분리
7. 타입 힌트를 통한 입출력 계약 명시
8. `argparse`를 이용한 하위 명령어 기반 CLI 설계
9. CSV 파일 가져오기와 내보내기
10. 임시 파일과 원자적 교체를 이용한 데이터 보호
11. 예외 상황에서도 기존 데이터를 안전하게 보존하는 방법
