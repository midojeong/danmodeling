# Document

## 주요 테이블
- course
- schedule
- student
- enroll
- teacher
- session
- payment
- invoice

## API endpoint
request body의 포맷은 JSON으로  `{ data: { ...payload } }` 이다. CRUD API에서
R(read)는 get이라는 명칭으로 통일하며, 이 때 메소드또한 get이 된다. 따라서 parameter들은
쿼리 스트링으로 넣어준다 e.g. `GET /api/v1/course/get?sort=createdAt&by=increase&size=30&page=0`

리스트로 받아와야 하는 경우 위의 방식처럼 page=0&size=30의 페이지네이션을 추가함. (이 때는
검색된 결과의 리스트에서 0부터 29번째 아이템까지 전송.

모든 table에 대해서 기본 get 메소드가 존재하는데 이는 `GET /api/v1/:테이블명/:id`의 형식을 따른다.

### course

- GET /:id
    - 요청: 
    - 응답: `{ id: ..., name: .. }`
    - 설명: 기본 GET 메소드

- GET /get?sort=......
    - 요청: 
    - 응답: `[ course1, course2, ... ]`
    - 설명: 여러 course를 한꺼번에 받아오는 메소드. 페이지네이션이 들어가있음

- GET /get/all?visible=false
    - 요청: 
    - 응답: `[ course1, course2, ... courseN ]`
    - 설명: 모든 course를 다 받아오는 메소드. 쿼리 visibile field에 true가 들어가면 숨김처리 된 course들까지 다 가져옴.

- GET /get/students/:id
    - 요청: 
    - 응답: `[ student1, student2, ... studentN ]`
    - 설명: 해당 수업에 등록된 학생들을 받아오는 메소드. 서버 내부에서 course id로 enroll table을 검색하고 student를 join해서 줘야함. 

- GET /get/schedules/:id
    - 요청: 
    - 응답: `[ schedule1, schedule2, ... scheduleN ]`
    - 설명: 해당 수업의 모든 스케쥴을 받아옴

- POST /search/name
    - 요청: `{ name: 수업명 }`
    - 응답: `[ course1, course2, ... courseN ]`
    - 설명: 수업명을 검색해서 일치하는 수업들만 보내줌( 당연히 중복될 수 있기 때문에 리스트로 ). 이 메소드는 프론트 UI상에서 어떤 수업을 선택할 때,
           사용자가 이름으로 검색한 뒤 선택하는 방식으로 각종 요청들을 처리하기 위해 사용. 당분간은 모든 course를 `/get/all?visible=false`
           로 받아와서 프론트상에서 search를 해줄 것이기 때문에 중요도 낮음 (차근차근 개발)
    
- POST /create
    - 요청: `{ name: 수업명, price: 가격, teacher: 선생id, detail: 설명 }`
    - 응답: course
    - 설명: 만들어진 course를 그대로 보내줌. (id도 포함)

- POST /update/schedule/:id  **(매우 중요)**
    - 요청: ```[ 
     {mode: "create", from: null , to: [날짜, 시간] }, 
     {mode: "create", from: null, to: [날짜, 시간]},
     {mode: "update", from: 스케쥴id, to: [새날짜, 새시간]}
     {mode: "delete", from: 스케쥴id, to: null}
     ...
    ]```
    - 응답: `[ schedule1, schedule2, schedule3, .... ]` 변경된 course의 모든 스케쥴을 다 보내주면됨. 실패코드는 아래에 설명
    - 설명: 이미 존재하는 schedule의 time이 바뀌면(`{mode: update, ...}` 그 schedule의 id를 들고있는 (fk_schdeule 이 있는) 세션들의 attendance가 업데이트됨(time이 바뀔 경우) 당연히 session의 charged 및 net도 변경될 것임. 새로운 스케쥴들이 생성되는 경우(`{mode: create ...}`) course에 등록된 학생을 검색(enroll table에서). 세션을 생성해줘야함. 스케쥴을 삭제하는 경우 (`{mode: delete}`) 해당 스케쥴과 연결된 session들은 모두 `active: false` 처리가 되며, charged도 0으로 바뀌고 net은 -paid로 바뀜. 이에 맞춰서 session들의 paymentState도 바뀌어야함. 즉 모든 경우(mode: create, delete, update)에 있어서 연관되는 session들을 생성, 삭제, 업데이트해줘야 함.
    
    - 제한조건: 위 요청을 처리한 결과로 만들어지는 날짜들이 겹치면 안됨(**중요**) 겹치게 될 경우, 코드 401, 없는 id의 스케쥴을 delete로 지우려할 경우 코드 402의 예외처리(더 있겠지만 기본적으로는). 프론트에서도 요청을 생성하기전에 한번 검수할 것이긴 하지만, 여럿이서 동시접근했을 때 db에서 에러가 나지 않으려면 백엔드에서 이렇게 한번 필터링을 해줘야함.
    


- POST /update/price/:id  **(중요)**
    - 요청: `{ price: ... }`
    - 응답: 200(성공) 404(course가 존재하지 않을 때)
    - 설명: course의 가격을 변경하는 메소드. 서버에서는 내부적으로 하위 스케쥴 및 세션들을 검색해서 업데이트시켜줘야함.
    
- POST /update/teacher/:id
    - 요청: `{teacher: 선생님ID, contract: 계약금, contractType: 계약타입}`
    - 응답: 200(성공) 401(선생님이 존재하지 않을 때)
    - 설명: 수업에 선생을 등록하는 과정. 이 때 계약금과 계약타입을 course.cotract, course.contractType에 업데이트.
           이미 등록된 선생이 현재 수업에 대해서만 계약내용을 변경하고싶어도 이 메소드를 사용함. 
           (그러나 선생의 계약조건이 업데이트된다고해서 자동으로 특정 수업의 계약조건이 업데이트되진 않음)
           (현재 erd및 DB모델링 상에는 course에 teacher외래키만 있을 뿐 contract, contractType은 없는데, 새로 추가해야함. null가능)

- DELETE /delete/:id
    - 요청: ``
    - 응답: todo
    - 설명: todo

### student

- GET /:id
    - 요청: 
    - 응답: `{ id: ..., name: .. }`
    - 설명: 기본 get 메소드

- GET /get?sort=......
    - 요청: 
    - 응답: `[ student1, student2, ... ]`
    - 설명: 여러 student를 한꺼번에 받아오는 메소드. 페이지네이션이 들어가있음

- GET /get/all?visible=false
    - 요청: 
    - 응답: `[ student1, student2, ... studentN ]`
    - 설명: 모든 student를 다 받아오는 메소드. 쿼리 visibile field에 true가 들어가면 숨김처리 된 student들까지 다 가져옴.

- GET /get/courses/:id
    - 요청: 
    - 응답: `[ course1, course2, ... courseN ]`
    - 설명: 해당 학생이 듣는 모든 수업을 받아옴

- GET /get/invoices/:id
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... invoiceN ]`
    - 설명: 학생에 포함된 모든 invoice(영수증)을 받아옴
    
- GET /get/sessions/:id?startAt=시작일endAt=종료일
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... invoiceN ]`
    - 설명: 학생이 등록된 수업의 세션들을 가져오는데, 시작일과 종료일을 쿼리 파라미터로 지정한다. 
           즉, 이 학생이 오늘 듣는 세션을 가져오고 싶으면 startAt=오늘0시0분&endAt=내일0시0분으로 지정. **날짜 포맷은 어떻게할까?**
           그러나 날짜 정보가 session에는 포함되어있지 않으므로(session과 연결된 schedule에 있음) 내부적으로 연산이좀 필요.

- POST /search/name
    - 요청: `{ name: 학생이름 }`
    - 응답: `[ student1, student2, ... studentn ]`
    - 설명: 학생 이름을 검색해서 가져옴. course/search/name과 마찬가지로 중요도 낮음.
    
- POST /create
    - 요청: `{ name: 학생이름, discount: 할인, discountReason: 할인근거, age: 나이, phone: 전화번호, detail: 설명 }`
    - 응답: student
    - 설명: 만들어진 student를 그대로 보내줌. (id도 포함)

- POST /update/discount/:id
    - 요청: `{ discount: 할인, discount: 할인근거 }`
    - 응답: 200(성공) 404(student가 존재하지 않을 때)
    - 설명: 학생의 기본 할인율과 근거를 업데이트. **이 때** 학생이 듣고있는 세션을 업데이트하지 않는다. 부수효과없이 학생 정보만 업데이트.

- DELETE /delete/:id
    - 요청: ``
    - 응답: TODO
    - 설명: TODO

### schedule

schedule에는 create메소드가 외부로 노출되지 않는다. 왜냐하면 schedule은 course/update/schedule/:id로만 생성, 삭제, 변경할 수 있기 때문.
마찬가지로 update와 delete 메소드도 노출되지 않음.

- GET /:id
    - 요청: 
    - 응답: `{ id: ..., course: .. }`
    - 설명: 기본 get 메소드

- GET /get?sort=date&by=increase&size=30&page=0 ....
    - 요청: 
    - 응답: `[ schedule1, schedule2, ... ]`
    - 설명: 여러 schedule를 한꺼번에 받아오는 메소드. 페이지네이션이 들어가있음

- GET /get/all?visible=false
    - 요청: 
    - 응답: `[ schedule1, schedule2, ... scheduleN ]`
    - 설명: 모든 schedule를 다 받아오는 메소드. 쿼리 visibile field에 true가 들어가면 숨김처리 된 schedule들까지 다 가져옴.

- GET /get/sessions/:id
    - 요청: 
    - 응답: `[ session1, session2, ... sessionN ]`
    - 설명: 해당 스케쥴과 연결된 모든 session을 보여줌.

- GET /get/date?startAt=시작일&endAt=종료일
    - 요청: 
    - 응답: `[ schedule1, schedule2, ... ]`
    - 설명: 여러 schedule를 한꺼번에 받아오는 메소드.

### enroll

- POST /create
    - 요청: `{ course: COURSE_ID, student: STUDENT_ID }`
    - 응답: code 200(성공) 400(course나 student가 존재하지않을 때) 401(이미 등록되어있을 때) 
    - 설명: 학생을 수업애 등록하는 API.
  
- DELETE /delete
    - 요청: `{ course: COURSE_ID, student: STUDENT_ID }`
    - 응답: 200(성공) 400(course나 student가 존재하지 않을 때, 401(수업에 등록되어있지 않을 때)
    - 설명: 학생을 수업에서 제외시키는 API. 

### teacher

- GET /:id
    - 요청: 
    - 응답: `{ id: ..., name: .. }`
    - 설명: 기본 get 메소드

- GET /get?sort=name&by=...
    - 요청: 
    - 응답: `[ teacher1, teacher2, ... ]`
    - 설명: 여러 teacher를 한꺼번에 받아오는 메소드. 페이지네이션이 들어가있음

- GET /get/all?visible=false
    - 요청: 
    - 응답: `[ teacher1, teacher2, ... teacherN ]`
    - 설명: 모든 teacher를 다 받아오는 메소드. 쿼리 visibile field에 true가 들어가면 숨김처리 된 teacher들까지 다 가져옴.

- GET /get/course/:id
    - 요청: 
    - 응답: `[ course1, course2, ... courseN ]`
    - 설명: 선생이 가르치는 모든 course를 가져옴.

- POST /search/name
    - 요청: `{ name: 선생이름 }`
    - 응답: `[ teacher1, teacher2, ... teacherN ]`
    - 설명: 선생 이름을 검색해서 가져옴. course/search/name과 마찬가지로 중요도 낮음.
    
- POST /create
    - 요청: `{ name: 이름, contract: 계약금액, contractType: 계약종류, detail: 설명 }`
    - 응답: teacher
    - 설명: 만들어진 teacher를 그대로 보내줌. (id도 포함)

- POST /update/contract/:id
    - 요청: `{  contract: 계약금액, contractType: 계약종류 }`
    - 응답: teacher
    - 설명: 선생의 기본 계약금, 타입을 업데이트함 teacher를 그대로 보내줌. (id도 포함)

- DELETE /delete/:id
    - 요청: ``
    - 응답: TODO
    - 설명: TODO

### session

session도 마찬가지로 외부로 공개된 create와 delete메소드는 없음. 그러나 update메소드들은 존재.
session은 get/all을 할 수 없음 (너무 많기 때문에)
그리고 보통의 get 메소드들의 종류도 많지 않은데 그 이유는 session에서 참조하고있는 course, student, schedule의
get메소드들에서 join해서 받아올 것이기 때문.

- GET /:id
    - 요청: 
    - 응답: `해당 id가진 세션`
    - 설명: 기본 get 메소드

- POST /get/id
    - 요청: `[ 세션아이디1, 세션아이디2, ... ]`
    - 응답: `[ session1, session2, ... ]`
    - 설명: 세션 아이디들을 보내면 그대로 세션 데이터를 넣어서 보내주면 됨. 사실 대부분의 get은 위에서 말한 다른 메소드들이
           처리할 예정이지만, 확장가능성을 위해 만들어두는 메소드(중요도 낮음)

- GET /get/payments/:id
    - 요청: 
    - 응답: `[ payment1, payment2, ... paymentN ]`
    - 설명: 해당 세션의 payment를 가져옴 (일종의 invoice상세 히스토리)
    
- POST /update/discount/:id
    - 요청: `{  discount: 할인, discountReason: 할인근거 }`
    - 응답: session
    - 설명: 해당 세션의 기본 할인 및, 근거를 업데이트함 session를 그대로 보내줌. (id도 포함). 이때 서버 내부적으로는 
      session의 charged, net, paid, paymentState가 변경됨.

- POST /update/attendance/:id
    - 요청: `{ attendance: 출석타임수 }`
    - 응답: session
    - 설명: 수업의 출석을 변경. session를 그대로 보내줌. (id도 포함) 이때 서버 내부적으로는 
      session의 charged, net, paid, paymentState가 변경됨.

- POST /update/active/:id
    - 요청: `{ active: true or false  }`
    - 응답: session
    - 설명: 수업의 활성화 상태를 변경. session를 그대로 보내줌. (id도 포함) active가 false가 되면 서버 내부적으로는 
      charged는 0이 되고, net은 paid-charged 이기 때문에 -charged로 변경됨 동시에 paymentState는 NEED_TO_BE_REFUND로
      변함 (즉 환불이 필요한 session이 됨). 만약 비활성화시킨 세션을 다시 활성화 시키는 경우는 charged를 강제로 0으로 만들어주지
      않기 때문에 charged, paid, net이 다시 계산되고 paymentState도 이 에 따라 변경

### payment

create, delete메소드가 따로 존재하지 않음. create는 invoice/create에서 처리해주고
delete도 마찬가지로 invoice/delete가 처리함. get도 기본메소드 뺴고는 존재하지 않는데, 그 이유는
invoice와 session에서 각각 받아가기 떄문임.

- GET /:id
    - 요청: 
    - 응답: `{ id, amount, reason, student, ...  }`
    - 설명: 기본 GET 메소드

- POST /update/reason/:id
    - 요청: `{ reason: 다른수업으로 이전 수강? 환불? 결제? }`
    - 응답: payment
    - 설명: 어차피 amount는 고정값, invoice와 schedule fkey도 고정값이기 때문에 바꿀 수 있는건 reason밖에 없음

### invoice

- GET /:id
    - 요청: 
    - 응답: `{ id, student, amount, ...  }`
    - 설명: 기본 GET 메소드

- GET /get?sort=......
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... ]`
    - 설명: 여러 invoice를 한꺼번에 받아오는 메소드. 페이지네이션이 들어가있음

- GET /get/all?visible=false
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... invoiceN ]`
    - 설명: 모든 invoice를 다 받아오는 메소드. 쿼리 visibile field에 true가 들어가면 숨김처리 된 invoice들까지 다 가져옴.

- GET /search/createdAt?startAt=시작일&endAt=종료일
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... ]`
    - 설명: createdAt(인보이스가 만들어진 날짜)를 기준으로 날짜 범위를 지정해서 탐색.

- GET /search/settledAt?startAt=시작일&endAt=종료일
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... ]`
    - 설명: settledAt(결제가 이루어진 날짜)를 기준으로 날짜 범위를 지정해서 탐색.

- GET /search/publishedAt?startAt=시작일&endAt=종료일
    - 요청: 
    - 응답: `[ invoice1, invoice2, ... ]`
    - 설명: publishedAt(인보이스가 학부모에게 전송된 날짜)를 기준으로 날짜 범위를 지정해서 탐색.

- GET /get/payment/:id
    - 요청: 
    - 응답: `[ payment1, payment2, ... paymentN ]`
    - 설명: invoice 가지고있는 모든 payment를 받아옴.
    
- POST /create **(매우 중요)**
    - 요청: ```
    { 
      sessions: [ [session1, reason1], [session2, reason2], ... [sessionN, reasonN]], 
      money: { card: 카드로 지불하는 금액, cash: ..., transfer },
      detail: ...,
      student: 학생ID,
    }```
    - 응답: invoice
    - 설명: 만들어진 invoice를 그대로 보내줌. (id도 포함). reason이 포함되는 이유는 이를 사용해서 바로 payment들을 생성해줘야 하기 때문.

- POST /update/money/:id
    - 요청: `{ card: 카드로 지불하는 금액, cash: ..., transfer, cardSettled: true|false, transferSettled: true|false }`
    - 응답: invoice
    - 설명: 지불 방식과, 실제 돈이 들어온 유무에 대해서 업데이트

- POST /update/published/:id
    - 요청: `{ published: 부모에게 영수증을 전달했을 때(true|false), publishedAt: 전달한 날짜 }` **설명 참고**
    - 응답: invoice
    - 설명: `publishedAt` 필드는 서버에서 자동으로 생성해주지 않으며, 클라에서 넣어준다. 그 이유는 전달된 날짜를 변경해야할 일이 생길 수도 있기 때문.

- POST /update/settled/:id
    - 요청: `{ settled: 결제가 됐을 때(true|false), settledAt: 결제 날짜 }` **설명 참고**
    - 응답: invoice
    - 설명: `settledAt` 필드는 서버에서 자동으로 생성해주지 않으며, 클라에서 넣어준다. 그 이유는 전달된 날짜를 변경해야할 일이 생길 수도 있기 때문.

- DELETE /delete/:id
    - 요청: ``
    - 응답: todo
    - 설명: todo
