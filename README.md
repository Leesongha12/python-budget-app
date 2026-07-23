# Python Console Budget App

Python 표준 라이브러리만 사용하여 구현한 파일 기반 콘솔 가계부 애플리케이션입니다.

거래 추가·조회·검색·수정·삭제뿐 아니라 월별 요약, 예산 관리, 카테고리 관리, CSV 가져오기·내보내기를 지원합니다.

이 프로젝트는 단순한 데이터 저장 프로그램이 아니라 다음 원칙을 적용한 유지보수 가능한 작은 서비스를 목표로 합니다.

* 모델, 저장소, 서비스, CLI 계층 분리
* JSONL 기반 영구 저장
* 거래·카테고리·예산 파일 분리
* 제너레이터 기반 스트리밍 조회
* 임시 파일과 `os.replace()`를 이용한 원자적 파일 교체
* 데코레이터를 이용한 예외 처리·로그·실행 시간 측정
* 타입 힌트를 이용한 계층 간 입출력 계약 명시
* 사용자에게 스택트레이스 대신 원인과 해결 방법 제공
* CSV import 일부 실패 시 정상 행은 저장하고 오류 행은 기록하는 부분 성공 정책

---

## 1. 주요 기능

애플리케이션은 다음 명령을 제공합니다.

```text
add → 거래 추가
list → 최신순 거래 목록 조회
search → 기간·카테고리·타입·메모·태그 조건 검색
summary → 월별 총수입·총지출·잔액·카테고리별 지출 요약
budget → 월별 예산 설정·조회
category → 카테고리 추가·목록·삭제
update → 거래 ID 기반 수정
delete → 거래 ID 기반 삭제
import → CSV 거래 일괄 등록
export → 월 또는 기간 조건의 거래 CSV 내보내기
```

전체 기능을 연속적으로 실행하는 예시는 다음과 같습니다.

```bash
python -m budget_app category list
python -m budget_app category add --name subscription
python -m budget_app add
python -m budget_app list --limit 10
python -m budget_app search --from 2024-01-01 --to 2024-01-31 --type expense
python -m budget_app summary --month 2024-01 --top 3
python -m budget_app budget set --month 2024-01 --amount 500000
python -m budget_app update --id TX-000001 --amount 18000
python -m budget_app delete --id TX-000002
python -m budget_app export --out exports/2024-01.csv --month 2024-01
python -m budget_app import --from import.csv
```

---

## 2. 개발 환경

* Python 3.10 이상
* Python 표준 라이브러리만 사용
* 별도 외부 패키지 설치 불필요

Python 버전을 확인합니다.

```bash
python --version
```

환경에 따라 다음 명령을 사용할 수 있습니다.

```bash
python3 --version
```

---

## 3. 프로젝트 구조

```text
python-budget-app/
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
│   ├── budget_app.log
│   └── import_errors.log
│
└── README.md
```

---

## 4. 모듈별 책임과 인터페이스

| 모듈                | 주요 클래스·함수                                                | 책임                        |
| ----------------- | -------------------------------------------------------- | ------------------------- |
| `__main__.py`     | `main() -> int`                                          | 프로그램 실행 진입점과 종료 코드 반환     |
| `cli.py`          | `create_parser()`, `run_command()`                       | 명령어·옵션 파싱, 대화형 입력, 결과 출력  |
| `models.py`       | `Transaction`, `Budget`, `MonthlySummary`                | 데이터 구조, 직렬화·역직렬화          |
| `repositories.py` | `TransactionRepository`, `CategoryStore`, `BudgetStore`  | JSONL 파일 읽기·쓰기·검색·원자적 재작성 |
| `services.py`     | `TransactionService`, `BudgetService`, `CategoryService` | 입력 검증 이후의 비즈니스 규칙 수행      |
| `decorators.py`   | `handle_cli_errors`, `log_execution`, `measure_time`     | 공통 예외 처리, 로그 기록, 실행 시간 측정 |
| `validators.py`   | `validate_date`, `validate_amount`, `validate_month`     | 날짜·금액·타입·월 형식 검증          |
| `formatter.py`    | `format_transaction`, `format_summary`                   | 콘솔 테이블과 금액 출력 형식 담당       |

계층 간 대표 호출 흐름은 다음과 같습니다.

```text
CLI
  ↓ 사용자 입력과 argparse.Namespace
Service
  ↓ 검증된 Transaction 또는 검색 조건
Repository
  ↓ JSONL 파일 스트리밍 및 영구 저장
Model
  ↓ 직렬화·역직렬화된 데이터 객체
```

CLI는 파일에 직접 접근하지 않고 서비스를 호출합니다.

서비스는 저장 형식을 직접 다루지 않고 저장소 인터페이스를 호출합니다.

저장소는 출력 문구나 사용자 입력을 처리하지 않고 파일 입출력만 담당합니다.

---

## 5. 데이터 모델

### Transaction

거래 한 건을 표현합니다.

```python
@dataclass
class Transaction:
    id: str
    type: Literal["income", "expense"]
    date: str
    amount: int
    category: str
    memo: str = ""
    tags: list[str] = field(default_factory=list)
```

핵심 책임:

* 거래 필드 보관
* JSON 객체로 직렬화
* JSON 객체에서 역직렬화
* 거래 타입과 필수 필드 구조 명시

### Budget

월별 예산을 표현합니다.

```python
@dataclass
class Budget:
    month: str
    amount: int
```

핵심 책임:

* `YYYY-MM` 단위 월 정보 보관
* 월별 예산 금액 보관
* 예산 JSONL 직렬화·역직렬화

### MonthlySummary

월별 계산 결과를 표현합니다.

```python
@dataclass
class MonthlySummary:
    month: str
    total_income: int
    total_expense: int
    balance: int
    category_expenses: dict[str, int]
    budget: int | None
```

핵심 책임:

* 총수입·총지출·잔액 계산 결과 전달
* 카테고리별 지출 합계 전달
* 예산 사용률과 초과 여부 계산에 필요한 값 제공

---

## 6. 실행 방법

프로젝트 최상위 디렉터리에서 실행합니다.

```bash
python -m budget_app <command> [options]
```

전체 도움말:

```bash
python -m budget_app --help
```

명령별 도움말:

```bash
python -m budget_app add --help
python -m budget_app list --help
python -m budget_app search --help
python -m budget_app summary --help
python -m budget_app update --help
python -m budget_app delete --help
python -m budget_app import --help
python -m budget_app export --help
```

---

## 7. 데이터 저장 위치

기본 데이터 폴더는 프로젝트 루트의 `./data`입니다.

```text
data/
├── transactions.jsonl
├── categories.jsonl
└── budgets.jsonl
```

다른 저장 위치를 사용하려면 전역 옵션 `--data-dir`을 지정합니다.

```bash
python -m budget_app --data-dir ./my_data list
```

지정한 폴더가 없으면 자동으로 생성합니다.

```text
my_data/
├── transactions.jsonl
├── categories.jsonl
└── budgets.jsonl
```

---

## 8. 최초 실행과 기본 카테고리

저장 폴더나 저장 파일이 없으면 프로그램이 자동으로 생성합니다.

카테고리 파일이 비어 있으면 다음 기본 카테고리가 자동 등록됩니다.

```text
food
transport
rent
salary
shopping
medical
education
entertainment
utilities
etc
```

초기화 예시:

```text
[초기화] data/transactions.jsonl 파일을 생성했습니다.
[초기화] data/categories.jsonl 파일을 생성했습니다.
[초기화] data/budgets.jsonl 파일을 생성했습니다.
[초기화] 기본 카테고리 10개를 등록했습니다.
```

---

## 9. 재실행 후 데이터 유지 확인

모든 데이터는 JSONL 파일에 영구 저장되므로 프로그램을 종료한 뒤 다시 실행해도 유지됩니다.

첫 번째 실행:

```bash
python -m budget_app add
```

```text
날짜(YYYY-MM-DD): 2024-01-15
타입(income/expense): expense
카테고리: food
금액(양수 정수): 15000
메모(선택): 점심
태그(쉼표로 구분, 없으면 엔터): meal
[저장 완료] id=TX-000001
```

프로그램을 종료한 뒤 다시 실행합니다.

```bash
python -m budget_app list --limit 1
```

```text
TX-000001 | 2024-01-15 | expense | food | 15,000원 | 점심
```

실행 전후에 같은 `data/transactions.jsonl` 파일을 사용하므로 거래가 유지됩니다.

---

# 거래 관리

## 10. 거래 추가

`add`는 대화형 입력 방식으로 실행합니다.

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

| 필드         | 필수 | 설명                    |
| ---------- | -- | --------------------- |
| `id`       | 자동 | 유일한 거래 ID             |
| `date`     | Y  | `YYYY-MM-DD`          |
| `type`     | Y  | `income` 또는 `expense` |
| `category` | Y  | 등록된 카테고리              |
| `amount`   | Y  | 0보다 큰 정수              |
| `memo`     | N  | 거래 설명                 |
| `tags`     | N  | 태그 목록                 |

거래 ID는 다음과 같이 자동 생성됩니다.

```text
TX-000001
TX-000002
TX-000003
```

삭제된 ID는 다시 사용하지 않습니다.

---

## 11. 거래 목록

최신 거래부터 출력합니다.

```bash
python -m budget_app list
```

기본 출력 건수는 20건입니다.

```bash
python -m budget_app list --limit 3
```

출력 예시:

```text
TX-000012 | 2024-01-15 | expense | food      |    15,000원 | 점심
TX-000011 | 2024-01-14 | income  | salary    | 3,000,000원 |
TX-000010 | 2024-01-12 | expense | transport |    20,000원 |
```

파일 전체를 거래 객체 리스트로 변환하지 않고 제너레이터로 한 줄씩 읽습니다.

---

## 12. 거래 검색

검색 명령:

```bash
python -m budget_app search [options]
```

지원 조건:

| 옵션                  | 설명                    |
| ------------------- | --------------------- |
| `--from YYYY-MM-DD` | 시작일                   |
| `--to YYYY-MM-DD`   | 종료일                   |
| `--category NAME`   | 카테고리                  |
| `--type TYPE`       | `income` 또는 `expense` |
| `--q KEYWORD`       | 메모 검색어                |
| `--tag TAG`         | 태그                    |
| `--limit N`         | 최대 출력 건수              |

기간 검색:

```bash
python -m budget_app search --from 2024-01-01 --to 2024-01-31
```

카테고리 검색:

```bash
python -m budget_app search --category food
```

지출 검색:

```bash
python -m budget_app search --type expense
```

메모 검색:

```bash
python -m budget_app search --q 점심
```

태그 검색:

```bash
python -m budget_app search --tag meal
```

조건 조합:

```bash
python -m budget_app search \
  --from 2024-01-01 \
  --to 2024-01-31 \
  --category food \
  --type expense \
  --tag meal
```

결과가 없는 경우:

```text
[안내] 검색 조건에 맞는 거래가 없습니다.
```

---

## 13. 거래 수정

거래 수정은 **옵션 기반 방식**으로 고정합니다.

```bash
python -m budget_app update --id <거래 ID> [수정 옵션]
```

수정 가능한 옵션:

| 옵션                      | 설명       |
| ----------------------- | -------- |
| `--date YYYY-MM-DD`     | 날짜       |
| `--type income/expense` | 거래 타입    |
| `--category NAME`       | 카테고리     |
| `--amount NUMBER`       | 금액       |
| `--memo TEXT`           | 메모       |
| `--tags TAGS`           | 쉼표 구분 태그 |

금액 수정:

```bash
python -m budget_app update --id TX-000012 --amount 18000
```

여러 필드 수정:

```bash
python -m budget_app update \
  --id TX-000012 \
  --category food \
  --memo "팀 점심 식사" \
  --tags "meal,work,team"
```

성공:

```text
[수정 완료] id=TX-000012
```

없는 ID:

```text
[오류] id=TX-999999에 해당하는 거래가 없습니다.
[힌트] list 또는 search 명령으로 거래 ID를 확인해 주세요.
```

수정 필드가 없는 경우:

```text
[오류] 수정할 항목이 지정되지 않았습니다.
[힌트] --amount, --memo 등의 수정 옵션을 입력해 주세요.
```

---

## 14. 거래 삭제

```bash
python -m budget_app delete --id TX-000012
```

성공:

```text
[삭제 완료] id=TX-000012
```

없는 ID:

```text
[오류] id=TX-999999에 해당하는 거래가 없습니다.
[힌트] list 또는 search 명령으로 거래 ID를 확인해 주세요.
```

---

# 월별 요약과 예산

## 15. 월별 요약

```bash
python -m budget_app summary --month 2024-01
```

카테고리별 지출 상위 개수를 지정합니다.

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

지출 TOP 3
1. rent            150,000원
2. food             45,000원
3. transport        20,000원
```

계산식:

```text
총 수입 = 해당 월 type=income 거래의 amount 합계
총 지출 = 해당 월 type=expense 거래의 amount 합계
잔액 = 총 수입 - 총 지출
카테고리별 지출 = type=expense 거래만 카테고리별 합산
```

데이터가 없는 경우:

```text
[안내] 2024-01에는 등록된 거래가 없습니다.
```

---

## 16. 예산 설정과 조회

예산 설정:

```bash
python -m budget_app budget set --month 2024-01 --amount 500000
```

```text
[저장 완료] 2024-01 예산 500,000원
```

같은 월에 다시 설정하면 기존 값을 수정합니다.

```text
[수정 완료] 2024-01 예산 600,000원
```

특정 월 조회:

```bash
python -m budget_app budget get --month 2024-01
```

```text
2024-01 예산: 500,000원
```

전체 조회:

```bash
python -m budget_app budget list
```

---

## 17. 예산 사용률과 초과 경고

예산이 설정된 월의 요약:

```bash
python -m budget_app summary --month 2024-01 --top 3
```

```text
2024-01 월별 요약
────────────────────────────────
총 수입: 3,000,000원
총 지출:   215,000원
잔액:     2,785,000원

예산:       500,000원
사용률:          43.0%
남은 예산:  285,000원
```

예산 계산식:

```text
예산 사용률 = 해당 월 총 지출 ÷ 월 예산 × 100
남은 예산 = 월 예산 - 해당 월 총 지출
```

예산 계산에는 `expense` 거래만 포함하며 `income` 거래는 제외합니다.

예산 초과:

```text
예산:       500,000원
사용률:         112.0%
초과 금액:   60,000원

[경고] 이번 달 지출이 설정된 예산을 초과했습니다.
```

---

# 카테고리 관리

## 18. 카테고리 목록

```bash
python -m budget_app category list
```

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

## 19. 카테고리 추가

대화형 방식:

```bash
python -m budget_app category add
```

```text
카테고리명: subscription
[저장 완료] category=subscription
```

옵션 방식:

```bash
python -m budget_app category add --name subscription
```

중복 카테고리:

```text
[오류] category=subscription은 이미 등록되어 있습니다.
[힌트] category list 명령으로 현재 카테고리를 확인해 주세요.
```

카테고리명은 앞뒤 공백을 제거하고 소문자로 저장합니다.

---

## 20. 카테고리 삭제

```bash
python -m budget_app category remove --name subscription
```

사용 중이지 않은 경우:

```text
[삭제 완료] category=subscription
```

거래에서 사용 중인 경우:

```text
[오류] category=food를 사용하는 거래가 8건 존재합니다.
[힌트] search --category food로 사용처를 확인한 뒤 update 명령으로 다른 카테고리로 변경해 주세요.
```

권장 삭제 절차:

```bash
python -m budget_app search --category food
python -m budget_app update --id TX-000001 --category etc
python -m budget_app update --id TX-000004 --category etc
python -m budget_app category remove --name food
```

데이터 일관성을 위해 사용 중인 카테고리를 자동으로 다른 카테고리로 변경하지 않습니다.

---

# CSV 가져오기와 내보내기

## 21. CSV 스키마

import와 export는 UTF-8 인코딩과 헤더가 포함된 CSV 파일을 사용합니다.

| 열          | 필수 | 설명                    |
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

CSV에는 `id`를 작성하지 않습니다.

import 시 거래 ID를 새로 생성합니다.

---

## 22. CSV 가져오기

```bash
python -m budget_app import --from import.csv
```

전체 성공:

```text
[완료] imported=5, skipped=0
```

일부 행 실패:

```text
[경고] 4번째 행을 건너뜁니다: 등록되지 않은 카테고리입니다.
[완료] imported=4, skipped=1
[로그] 실패 행 상세 정보가 logs/import_errors.log에 기록되었습니다.
```

검증 항목:

* 필수 헤더 존재 여부
* UTF-8 디코딩 가능 여부
* 날짜 형식
* 거래 타입
* 금액이 양수 정수인지 여부
* 카테고리 등록 여부

등록되지 않은 카테고리:

```text
[경고] 4번째 행을 건너뜁니다: category=insurance가 등록되지 않았습니다.
[힌트] category add --name insurance 실행 후 다시 가져와 주세요.
```

---

## 23. 부분 import 정책과 보장 수준

import는 **전체 롤백 방식이 아닌 부분 성공 방식**을 사용합니다.

```text
정상 행 → 거래 파일에 저장
오류 행 → 저장하지 않고 skipped 건수에 포함
오류 원인 → logs/import_errors.log에 행 번호와 함께 기록
```

예시 로그:

```text
2024-01-31 15:20:11 | row=4 | category=insurance | error=unknown category
2024-01-31 15:20:11 | row=7 | amount=-3000 | error=amount must be positive
```

보장 수준:

* 검증을 통과한 행만 저장합니다.
* 실패 행 때문에 정상 행을 취소하지 않습니다.
* 실패 행의 번호와 원인을 로그에 남겨 재처리할 수 있게 합니다.
* 파일 쓰기 자체가 실패하면 해당 쓰기 작업을 오류로 종료합니다.
* 한 번 저장된 정상 행은 import 결과의 `imported` 건수에 포함됩니다.
* 동일 CSV를 다시 import하면 중복 거래가 생성될 수 있으므로 사용자가 결과를 확인해야 합니다.

부분 성공 예시:

```text
[완료] imported=48, skipped=2
[로그] logs/import_errors.log에서 실패 행을 확인해 주세요.
```

---

## 24. CSV 내보내기

내보내기에는 다음 조건 중 하나 이상이 필요합니다.

* `--month YYYY-MM`
* `--from YYYY-MM-DD --to YYYY-MM-DD`

월 단위:

```bash
python -m budget_app export \
  --out exports/2024-01.csv \
  --month 2024-01
```

```text
[완료] exports/2024-01.csv (12 records)
```

기간 단위:

```bash
python -m budget_app export \
  --out exports/first-quarter.csv \
  --from 2024-01-01 \
  --to 2024-03-31
```

추가 조건:

```bash
python -m budget_app export \
  --out exports/food-expense.csv \
  --month 2024-01 \
  --category food \
  --type expense
```

조건이 없는 경우:

```text
[오류] 내보내기 범위가 지정되지 않았습니다.
[힌트] --month 또는 --from과 --to 옵션을 입력해 주세요.
```

결과가 0건이어도 헤더가 포함된 파일을 생성합니다.

```text
[완료] exports/empty.csv (0 records)
```

파일 생성 실패:

```text
[오류] exports/2024-01.csv 파일을 생성할 수 없습니다.
[힌트] 출력 폴더의 존재 여부, 파일 권한, 같은 이름의 파일이 사용 중인지 확인해 주세요.
```

---

# 저장 파일 형식

## 25. 거래 파일

경로:

```text
data/transactions.jsonl
```

예시:

```json
{"id":"TX-000012","date":"2024-01-15","type":"expense","category":"food","amount":15000,"memo":"점심","tags":["meal","work"]}
{"id":"TX-000011","date":"2024-01-14","type":"income","category":"salary","amount":3000000,"memo":"","tags":[]}
```

---

## 26. 카테고리 파일

경로:

```text
data/categories.jsonl
```

예시:

```json
{"name":"food"}
{"name":"transport"}
{"name":"rent"}
{"name":"salary"}
```

---

## 27. 예산 파일

경로:

```text
data/budgets.jsonl
```

예시:

```json
{"month":"2024-01","amount":500000}
{"month":"2024-02","amount":600000}
```

동일한 월에는 하나의 예산만 존재합니다.

같은 월의 예산을 다시 설정하면 기존 값을 변경합니다.

---

# JSONL 선택 근거

## 28. JSONL과 CSV 비교

| 항목         | JSONL                          | CSV                       |
| ---------- | ------------------------------ | ------------------------- |
| 데이터 구조     | 객체와 배열 표현에 적합                  | 행과 열 형태의 평면 데이터에 적합       |
| 태그 저장      | `["meal", "work"]`처럼 배열로 저장 가능 | 쉼표 구분 문자열로 별도 변환 필요       |
| 타입 보존      | 숫자, 문자열, 배열을 구분 가능             | 모든 값이 문자 형태로 읽힐 수 있음      |
| 한 줄 단위 처리  | 한 줄이 하나의 JSON 객체               | 한 행이 하나의 레코드              |
| 스트리밍       | 줄 단위 스트리밍이 쉬움                  | `csv.DictReader`로 스트리밍 가능 |
| 사람이 직접 수정  | 문법에 익숙하지 않으면 어려움               | 스프레드시트에서 수정하기 쉬움          |
| 외부 프로그램 호환 | 일부 도구에서 추가 변환 필요               | 엑셀 등과 호환성이 높음             |
| 필드 확장      | 새 필드 추가가 비교적 유연함               | 헤더 스키마 변경 필요              |
| 중첩 데이터     | 표현 가능                          | 표현하기 어려움                  |

### 본 프로젝트에서 JSONL을 선택한 이유

거래 데이터의 `tags`는 여러 값을 가지는 리스트이므로 JSON 구조가 자연스럽습니다.

또한 거래 한 건을 한 줄에 저장할 수 있어 다음 작업에 적합합니다.

* 거래 추가 시 파일 끝에 한 줄 추가
* 제너레이터로 한 줄씩 순차 읽기
* 객체의 숫자·문자열·배열 타입 보존
* 이후 새로운 필드를 추가할 때 비교적 유연한 확장

따라서 애플리케이션 내부 영구 저장에는 JSONL을 사용합니다.

반면 사용자와 외부 프로그램이 데이터를 교환하는 import/export 기능에는 호환성이 높은 CSV를 사용합니다.

```text
내부 영구 저장: JSONL
외부 데이터 교환: CSV
```

---

# 제너레이터 스트리밍

## 29. 거래 스트리밍

거래 파일은 전체 내용을 리스트로 반환하지 않고 한 줄씩 처리합니다.

```python
from collections.abc import Iterator

def iter_transactions(self) -> Iterator[Transaction]:
    with self.path.open("r", encoding="utf-8") as file:
        for line_number, line in enumerate(file, start=1):
            if not line.strip():
                continue

            yield Transaction.from_json(line)
```

장점:

* 거래 수가 증가해도 전체 데이터를 한 번에 메모리에 적재하지 않음
* 검색 조건에 맞지 않는 데이터는 즉시 버릴 수 있음
* summary와 export에서도 같은 스트림 구조를 재사용 가능
* 첫 번째 결과를 전체 로딩 완료 전부터 처리 가능

---

## 30. 순방향·역방향 처리 기준

기본 JSONL 저장은 오래된 거래부터 파일 뒤쪽으로 추가되는 구조입니다.

```text
파일 앞부분 → 오래된 거래
파일 뒷부분 → 최신 거래
```

처리 기준:

* `summary`, `search`, `export`는 전체 범위를 검사해야 하므로 순방향 스트리밍을 사용합니다.
* `list --limit N`은 최신 거래가 필요하므로 파일 뒤쪽부터 읽는 역방향 줄 읽기를 적용하거나, 순방향 스트림을 읽으면서 `deque(maxlen=N)`에 최근 N건만 유지합니다.
* 검색 결과를 최신순으로 제한해야 할 때도 `deque(maxlen=N)`을 사용하면 전체 거래 객체를 저장하지 않고 결과 N건만 메모리에 유지할 수 있습니다.
* 제한 없이 모든 검색 결과를 완전한 최신순으로 출력해야 한다면 결과 정렬 비용이 발생하므로 임시 파일이나 별도 인덱스가 필요합니다.

예시:

```python
from collections import deque
from collections.abc import Iterable

def latest_transactions(
    transactions: Iterable[Transaction],
    limit: int,
) -> list[Transaction]:
    buffer: deque[Transaction] = deque(maxlen=limit)

    for transaction in transactions:
        buffer.append(transaction)

    return list(reversed(buffer))
```

이 방식은 전체 거래가 100,000건이어도 메모리에는 최대 `limit`건만 유지합니다.

---

# 원자적 수정과 삭제

## 31. 파일 재작성 절차

JSONL은 특정 줄을 파일 내부에서 직접 안전하게 수정하기 어렵습니다.

따라서 update와 delete는 다음 순서로 수행합니다.

```text
1. 원본 transactions.jsonl을 읽기 전용으로 연다.
2. 같은 폴더에 transactions.tmp를 생성한다.
3. 원본 데이터를 한 줄씩 읽는다.
4. 수정·삭제 조건을 적용해 임시 파일에 기록한다.
5. 임시 파일 flush를 수행한다.
6. 필요하면 os.fsync()로 디스크 기록을 요청한다.
7. 임시 파일이 정상적으로 존재하고 기록되었는지 확인한다.
8. os.replace()로 원본과 임시 파일을 원자적으로 교체한다.
```

개념 코드:

```python
def rewrite_atomically(
    source_path: Path,
    transform: Callable[[Transaction], Transaction | None],
) -> None:
    temp_path = source_path.with_suffix(".tmp")

    try:
        with source_path.open("r", encoding="utf-8") as source:
            with temp_path.open("w", encoding="utf-8") as temp:
                for line in source:
                    transaction = Transaction.from_json(line)
                    changed = transform(transaction)

                    if changed is not None:
                        temp.write(changed.to_json() + "\n")

                temp.flush()
                os.fsync(temp.fileno())

        if not temp_path.exists():
            raise StorageError("임시 파일이 생성되지 않았습니다.")

        os.replace(temp_path, source_path)

    except Exception:
        if temp_path.exists():
            temp_path.unlink()
        raise
```

---

## 32. 임시 파일 실패 시 복구 정책

임시 파일 작성 중 오류가 발생하면 원본 파일을 교체하지 않습니다.

체크포인트:

```text
- 임시 파일 생성 여부 확인
- 임시 파일 쓰기 완료 여부 확인
- flush 및 fsync 완료 여부 확인
- 대상 ID 발견 여부 확인
- os.replace 실행 전 원본 파일 존재 여부 확인
```

실패 시:

```text
1. os.replace()를 실행하지 않는다.
2. 기존 transactions.jsonl을 그대로 유지한다.
3. 남아 있는 transactions.tmp를 삭제한다.
4. 오류 원인을 logs/budget_app.log에 기록한다.
5. 사용자에게 원본 데이터가 보존되었음을 안내한다.
```

출력 예시:

```text
[오류] 거래 파일을 다시 작성하는 중 문제가 발생했습니다.
[힌트] 원본 데이터는 변경되지 않았습니다. 파일 권한과 저장 공간을 확인해 주세요.
```

로그 예시:

```text
2024-01-31 15:31:22 | ERROR | atomic rewrite failed | temp=transactions.tmp | original_preserved=true
```

---

# 데코레이터

## 33. 공통 관심사 분리

명령어 함수에 반복되는 예외 처리, 로그, 실행 시간 측정을 데코레이터로 분리합니다.

```python
@handle_cli_errors
@log_execution
@measure_time
def run_summary_command(args: argparse.Namespace) -> int:
    ...
```

### `handle_cli_errors`

예상 가능한 애플리케이션 오류를 사용자 메시지와 종료 코드로 변환합니다.

```text
[오류] 날짜 형식이 올바르지 않습니다.
[힌트] YYYY-MM-DD 형식으로 입력해 주세요.
```

### `log_execution`

명령 시작, 성공, 실패 정보를 로그 파일에 기록합니다.

```text
2024-01-31 15:42:10 | INFO | command=summary | status=started
2024-01-31 15:42:10 | INFO | command=summary | status=success
```

오류 로그:

```text
2024-01-31 15:43:21 | ERROR | command=export | error=PermissionError
```

### `measure_time`

명령 실행 시간을 측정해 로그에 기록합니다.

```text
2024-01-31 15:42:10 | INFO | command=summary | elapsed_ms=12.45
```

사용자 화면에는 기본적으로 실행 시간을 출력하지 않고 로그에만 기록합니다.

---

# 타입 힌트와 계층 계약

## 34. 대표 타입 계약

거래 조회:

```python
def find_by_id(
    self,
    transaction_id: str,
) -> Transaction | None:
    ...
```

계약:

```text
입력: str 형식 거래 ID
출력: 거래가 있으면 Transaction, 없으면 None
```

사용 예시:

```python
transaction: Transaction | None = repository.find_by_id("TX-000001")

if transaction is None:
    raise NotFoundError("거래를 찾을 수 없습니다.")
```

월별 요약:

```python
def calculate_monthly_summary(
    self,
    month: str,
    top_n: int,
) -> MonthlySummary:
    ...
```

계약:

```text
입력: month="YYYY-MM", top_n=1 이상의 정수
출력: MonthlySummary 객체
오류: 월 형식이 잘못되면 ValidationError
```

거래 스트림:

```python
def iter_transactions(self) -> Iterator[Transaction]:
    ...
```

계약:

```text
입력: 없음
출력: Transaction 객체를 하나씩 생성하는 Iterator
오류: 손상된 JSONL 행이 있으면 StorageError
```

카테고리 목록:

```python
def list_categories(self) -> set[str]:
    ...
```

계약:

```text
입력: 없음
출력: 중복 없는 카테고리 문자열 집합
```

타입 힌트를 통해 CLI, 서비스, 저장소 계층이 주고받는 값의 형태를 명확하게 유지합니다.

---

# 오류 처리

## 35. 입력 오류

날짜 오류:

```text
[오류] 날짜 형식이 올바르지 않습니다.
[힌트] YYYY-MM-DD 형식으로 입력해 주세요. 예: 2024-01-15
```

금액 오류:

```text
[오류] 금액은 0보다 큰 정수여야 합니다.
[힌트] 쉼표나 단위를 제외하고 숫자만 입력해 주세요. 예: 15000
```

타입 오류:

```text
[오류] 허용되지 않은 거래 타입입니다.
[힌트] income 또는 expense 중 하나를 입력해 주세요.
```

카테고리 오류:

```text
[오류] category=insurance가 등록되어 있지 않습니다.
[힌트] category add --name insurance 명령으로 먼저 추가해 주세요.
```

---

## 36. 예상하지 못한 내부 오류

사용자 화면에는 스택트레이스를 출력하지 않습니다.

```text
[오류] 명령을 처리하는 중 예상하지 못한 문제가 발생했습니다.
[힌트] 원본 데이터는 보존되었습니다. logs/budget_app.log를 확인해 주세요.
```

상세 스택트레이스와 예외 정보는 개발자 확인용 로그 파일에 기록합니다.

```text
logs/budget_app.log
```

---

## 37. 손상된 JSONL 데이터

손상된 행을 발견하면 위치를 안내합니다.

```text
[오류] transactions.jsonl의 12번째 줄을 읽을 수 없습니다.
[힌트] 해당 줄의 JSON 형식을 확인하거나 백업 파일로 복구해 주세요.
```

데이터 손상 가능성이 있는 상태에서는 update와 delete를 중단하여 손상된 파일을 다시 덮어쓰지 않습니다.

---

# 종료 코드

## 38. 종료 코드 표

| 코드  | 의미               |
| --- | ---------------- |
| `0` | 정상 종료            |
| `1` | 일반 애플리케이션 오류     |
| `2` | 명령어·옵션 사용 오류     |
| `3` | 입력값 검증 오류        |
| `4` | 파일 읽기·쓰기 오류      |
| `5` | 요청한 데이터가 존재하지 않음 |

Linux·macOS:

```bash
python -m budget_app list --limit 3
echo $?
```

정상 실행 예시:

```text
TX-000003 | 2024-01-15 | expense | food | 15,000원 | 점심
0
```

오류 실행 예시:

```bash
python -m budget_app delete --id TX-999999
echo $?
```

```text
[오류] id=TX-999999에 해당하는 거래가 없습니다.
[힌트] list 또는 search 명령으로 거래 ID를 확인해 주세요.
5
```

Windows PowerShell:

```powershell
python -m budget_app list --limit 3
$LASTEXITCODE
```

---

# 대용량 데이터 분석

## 39. 100,000건 기준 예상 병목

거래가 약 100,000건으로 증가하면 JSONL 파일도 일반적인 개인 가계부 범위에서는 여전히 처리 가능하지만, 다음 작업에서 병목이 발생할 수 있습니다.

### 1. 전체 파일 순차 검색

`search`, `summary`, `export`는 조건에 맞는 거래를 찾기 위해 파일 전체를 읽습니다.

시간 복잡도:

```text
O(N)
```

100,000건 중 마지막 한 건을 찾더라도 앞의 모든 행을 읽어야 할 수 있습니다.

개선 방안:

* 월별 파일 분리
* 날짜별 바이트 오프셋 인덱스 생성
* 카테고리·월별 보조 인덱스 파일 생성
* 자주 사용하는 월별 집계 결과 캐시

예시 구조:

```text
data/
├── transactions/
│   ├── 2024-01.jsonl
│   ├── 2024-02.jsonl
│   └── 2024-03.jsonl
└── indexes/
    ├── month_index.json
    └── category_index.json
```

### 2. update와 delete의 전체 파일 재작성

한 건을 수정하거나 삭제해도 전체 거래 파일을 임시 파일로 다시 작성합니다.

시간 복잡도:

```text
O(N)
```

디스크 사용량도 원본 파일과 임시 파일이 동시에 존재하므로 일시적으로 약 2배 필요합니다.

개선 방안:

* 거래 파일을 월별로 분할해 해당 월 파일만 재작성
* 변경 로그 방식 적용
* 삭제 여부를 표시하는 tombstone 필드 사용
* 일정 횟수 이상 변경 로그가 쌓이면 compaction 수행
* 데이터 규모가 더 커지면 SQLite 등 인덱스를 지원하는 저장소로 전환

### 3. 최신순 전체 정렬

파일이 오래된 순서로 저장되어 있는데 검색 결과 전체를 최신순으로 출력하려면 결과를 메모리에 모아 정렬할 가능성이 있습니다.

시간·공간 복잡도:

```text
정렬 시간: O(K log K)
메모리: O(K)
```

`K`는 검색 조건에 맞는 결과 수입니다.

개선 방안:

* `--limit`이 있으면 `deque(maxlen=limit)` 사용
* 파일 역방향 읽기 적용
* 거래 저장 시 날짜 순서를 보장
* 월별 파일로 분리 후 필요한 월만 역순 순회
* 외부 정렬이 필요하면 임시 파일 기반 병합 정렬 적용

### 4. ID 생성

매번 가장 큰 ID를 찾기 위해 전체 거래 파일을 읽으면 거래 추가도 O(N)이 됩니다.

개선 방안:

```text
data/metadata.json
```

```json
{"last_transaction_id": 100000}
```

새 거래 추가 시 메타데이터의 마지막 ID만 읽고 증가시킵니다.

메타데이터 역시 임시 파일과 `os.replace()`로 안전하게 갱신합니다.

### 5. 반복 summary 계산

동일한 월을 여러 번 조회할 때마다 전체 파일을 읽으면 불필요한 반복 비용이 발생합니다.

개선 방안:

```text
data/summary_cache.json
```

월별 총수입·총지출·카테고리별 지출을 저장하고, 해당 월 거래가 추가·수정·삭제된 경우에만 캐시를 무효화합니다.

### 6. import 대량 처리

100,000행 CSV를 import하면서 거래마다 파일을 열고 닫으면 큰 병목이 발생합니다.

잘못된 방식:

```text
행 읽기 → 파일 열기 → 한 줄 쓰기 → 파일 닫기
```

개선 방식:

```text
파일 한 번 열기
→ CSV 행 스트리밍 검증
→ 정상 행을 연속 기록
→ 마지막에 flush와 fsync
→ 파일 닫기
```

### 7. 병렬 처리의 한계

단순 파일 기반 저장은 여러 프로세스가 동시에 같은 파일을 수정하면 충돌할 수 있습니다.

따라서 쓰기 작업을 무조건 병렬화하는 것은 안전하지 않습니다.

병렬 처리 적용 가능 영역:

* 읽기 전용 분석
* 서로 다른 월 파일 처리
* 독립된 CSV 검증 작업

병렬 처리 전 필요한 안전장치:

* 파일 잠금
* 월별 파일 분리
* 단일 writer 정책
* 충돌 감지
* 원자적 교체

---

## 40. 100,000건 처리 개선 우선순위

현재 과제 범위에서는 표준 라이브러리와 JSONL 구조를 유지하되 다음 순서로 개선할 수 있습니다.

```text
1순위: ID 메타데이터 파일 도입
2순위: list에 deque 또는 역방향 읽기 적용
3순위: 월별 JSONL 파일 분리
4순위: 월·카테고리 인덱스 생성
5순위: summary 캐시
6순위: 변경 로그와 compaction
7순위: 데이터 규모가 더 커지면 SQLite 전환
```

본 과제의 제너레이터 스트리밍은 메모리 사용량을 줄이는 데 효과적이지만, 전체 파일을 읽어야 하는 O(N) 탐색 시간까지 제거하지는 않습니다.

---

# 명령어 검증 시나리오

## 41. 전체 기능 실행 예시

### 카테고리 확인

```bash
python -m budget_app category list
```

### 거래 추가

```bash
python -m budget_app add
```

### 거래 목록 확인

```bash
python -m budget_app list --limit 5
```

### 거래 검색

```bash
python -m budget_app search --month 2024-01
```

기간 조건을 사용하는 구현에서는 다음과 같이 실행합니다.

```bash
python -m budget_app search --from 2024-01-01 --to 2024-01-31
```

### 예산 설정

```bash
python -m budget_app budget set --month 2024-01 --amount 500000
```

### 월별 요약

```bash
python -m budget_app summary --month 2024-01 --top 3
```

### 거래 수정

```bash
python -m budget_app update --id TX-000001 --amount 18000
```

### 거래 삭제

```bash
python -m budget_app delete --id TX-000002
```

### CSV 내보내기

```bash
python -m budget_app export --out exports/2024-01.csv --month 2024-01
```

### CSV 가져오기

```bash
python -m budget_app import --from import.csv
```

### 재실행 데이터 확인

```bash
python -m budget_app list --limit 5
```

---

# 기능 점검표

## 42. 필수 요구사항

* [ ] `add` 거래 추가
* [ ] `list` 최신순 거래 목록
* [ ] `search` 조건 검색
* [ ] `summary` 월별 요약
* [ ] `budget set/get/list`
* [ ] 예산 사용률과 초과 경고
* [ ] `category add/list/remove`
* [ ] 사용 중인 카테고리 삭제 차단
* [ ] `update --id`
* [ ] `delete --id`
* [ ] `import --from`
* [ ] `export --out`
* [ ] JSONL 영구 저장
* [ ] 거래·카테고리·예산 3개 이상 파일 분리
* [ ] 프로그램 재실행 후 데이터 유지
* [ ] 제너레이터 기반 스트리밍
* [ ] 데코레이터 1개 이상 실제 적용
* [ ] 타입 힌트 적용
* [ ] 3개 이상 모듈 분리
* [ ] 원자적 update/delete
* [ ] 스택트레이스 대신 오류 원인과 힌트 출력
* [ ] 정상 종료 코드 0
* [ ] 오류 종료 코드 비0
* [ ] 모든 명령의 `--help` 지원
* [ ] import 실패 행 로그 기록
* [ ] JSONL과 CSV 선택 근거 문서화
* [ ] 100,000건 기준 병목과 개선 방안 문서화

구현 완료 후 실제 실행 결과를 확인한 항목만 `[x]`로 변경합니다.

---

## 43. 핵심 학습 내용

이 프로젝트를 통해 다음 내용을 학습합니다.

1. JSONL 기반 데이터 영구 저장
2. 파일 기반 CRUD 구현
3. 제너레이터와 `yield`를 이용한 스트리밍
4. `dataclass` 기반 데이터 모델링
5. 모델·저장소·서비스·CLI 책임 분리
6. 데코레이터 기반 공통 관심사 분리
7. 타입 힌트를 통한 계층 간 계약 명시
8. `argparse` 하위 명령어 설계
9. CSV 가져오기·내보내기
10. 부분 import와 실패 행 기록
11. 임시 파일과 `os.replace()`를 이용한 데이터 보호
12. 종료 코드를 이용한 성공·실패 구분
13. JSONL과 CSV의 용도별 선택
14. 100,000건 규모에서의 시간·메모리·디스크 병목 분석
15. 파일 기반 저장소의 한계와 인덱스·캐시·SQLite 전환 기준
