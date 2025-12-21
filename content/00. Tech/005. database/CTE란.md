---
tags:
  - CTE
---
# 기본 문법

```sql
WITH cte_name AS (
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
SELECT *
FROM cte_name;
```

# 주요 특징

1. **가독성 향상** - 복잡한 쿼리를 단계별로 나눠 이해하기 쉽게 만듦
2. **재사용 가능** - 동일한 서브쿼리를 여러 번 참조 가능
3. **재귀 쿼리 지원** - 계층 구조 데이터 처리 가능

## 재귀 쿼리 예제

```sql
-- 기본 CTE
WITH sales_2024 AS (
    SELECT product_id, SUM(amount) as total_sales
    FROM orders
    WHERE YEAR(order_date) = 2024
    GROUP BY product_id
)
SELECT p.product_name, s.total_sales
FROM products p
JOIN sales_2024 s ON p.product_id = s.product_id;

-- 재귀 CTE
WITH RECURSIVE org_chart AS (
    -- 기본 케이스
    SELECT employee_id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- 재귀 케이스
    SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
    WHERE oc.level < 3  -- 3단계까지만 조회, 조회 계층의 깊이를 조절할 수 있음
)
SELECT * FROM org_chart;
```
### 작동 방식
#### 초기 실행 (기본 케이스)
```sql
SELECT employee_id, name, manager_id, 1 as level
FROM employees
WHERE manager_id IS NULL
```
- 최상위 매니저(manager_id가 NULL인 사람)를 먼저 조회
- 결과를 임시 테이블 `org_chart`에 저장
#### 재귀 실행(UNION ALL)
```sql
SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
FROM employees e
JOIN org_chart oc ON e.manager_id = oc.employee_id
```
- 이전 단계에서 나온 `org_chart` 결과를 기준으로 다시 조회
- **자동으로 반복**: 새로운 행이 나오지 않을 때까지 계속 실행
- 각 반복마다 결과를 `org_chart`에 추가
#### 종료 조건
- JOIN 조건(`e.manager_id = oc.employee_id`)을 만족하는 행이 더 이상 없으면 자동으로 멈춤
