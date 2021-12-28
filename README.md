# 수도권 원룸 가격 분석🏠

- [깃헙 주소](https://github.com/SuHae-Bae/BigDataProject)

## 개요

- 주제: 수도권(서울, 경기도, 인천) 지역의 원룸 가격 비교 및 분석
- 목표: 수도권의 원룸 데이터를 수집, 분석하여 어떤 지역이 가장 비싼지/저렴한지, 가장 매물이 많은지를 알아보고, 평당 가격(혹은 1제곱미터당 가격)이 가장 비싼 동네를 찾아본다(전세기준).
- 총 수집 데이터: 27MB
- 분석 대상 사이트: 네이버 부동산(https://land.naver.com/)
- 사용 기술: Python, Selenium, Chrome Webdriver, HDFS, Zeppelin



## 삽질의 역사...

1.처음에는 다른 깃허브에 있는 네이버 부동산 크롤링 작업물을 보고 참고하여 비슷하게 따라할 예정이었다.

그러나 이럴수가, 프로젝트 시작 3일만에 들려온 충격적인소식.

https://landad.naver.com/home/noticeView.naver?noti_mttr_seq=7147&tpprt_exps_yn=Y&search_type=&search=&page=1

![image](https://user-images.githubusercontent.com/33304926/147569667-abe9e15d-8696-46da-962b-097cbb126383.png)

그렇다

크롤링을 하려던 페이지가 페이지 개편을 한다는 공지였다.

그렇게 나는 3일만에 코드를 개편된 페이지 기준으로 뜯어고쳐야했다.

2.마침 수업에서 하둡을 배우고 있는 중이라 하둡의 Flume을 이용해 일정주기에 맞춰 크롤링을 해서 HDFS에 적재하려했으나,  웹드라이버가 하둡에서 돌아가지 않아서ㅠㅠㅠㅠㅠㅠㅠ로컬에서 윈도우의 작업 스케줄러를 이용해 매일 일정한 시간에 크롤러를 작동시켰다. 크아아아악 이게 됬으면 더 완벽한 파이프라인을 만들었을텐데!!!!!!!!

그 외 자잘한 삽질이 많았는데, 궁금하면 [작업삽질](https://github.com/SuHae-Bae/BigDataProject/blob/master/ProjectPlan/ProjectZakup_1.md)을 읽어보는것을 추천한다. 여기에 프로젝트를 진행하면서 있었던 고려사항, 해야하는 일들, 진행상황 등을 정말 간단하게 기록해두었다.



## 파이프라인&사이트 분석

- #### **파이프라인**

파이프라인은 정말 간단하다.

#### **매일 일정한 시간에 작업 스케줄러로 크롤러 작동  → 긁어온 데이터 정제 → Cyberduck과 Putty를 이용해 HDFS에 데이터 업로드) → zeppelin으로 데이터 분석 및 시각화**

구체적으로 설명하면 다음과 같다.

윈도우의 작업 스케줄러를 이용해 매일 오전 5:55분마다 크롤러 파일(Crawl_final.py)를 실행시키면 실행 결과가 TestCrawl_3_2.csv로 저장된다. 이 파일을 전처리 파일(Clean_final.py)를 통해 데이터 클렌징을 해 준다. 클렌징이 끝난 데이터는 TestClean_3_6.csv로 저장된다. 이 데이터를 Cyberduck와 Putty를 이용해 HDFS에 저장하여 Zeppelin으로 데이터 분석을 진행한다.

- #### **사이트 분석**

분석에 앞서, 몇몇 부동산 매물 사이트를 비교해보았다.

**kb부동산, 부동산 114**: 확인 가능한 매물의 양이 적음

**다방, 직방**: 실거래/거래완료 된 매물의 정보가 없음

**국토교통부 실거래가 공개시스템**: 실거래가 된 매물의 정보만 올라와서 데이터 양이 부족함.

**네이버 부동산**: 다양한 사이트에서 정보를 가져옴/등록,확인,실거래,거래완료된 매물 정보가 모두 올라와 있음

이런 이유로, 네이버 부동산 사이트를 채택하게 되었다.



![image](https://user-images.githubusercontent.com/33304926/147571012-1d0fbe82-e3a4-4638-aa7e-af3aea020273.png)

네이버 부동산 사이트는 다음과 같이 구성되어 있다.

①: 사이트 URL

②: 검색 옵션

③: 검색 결과

④: 지역 옵션

⑤: 지도 새로고침(목록 새로고침)



## 크롤러 분석

크롤러를 소개하기 앞서, 네이버 부동산 사이트의 구조(원룸기준)은 이렇다.

https://new.land.naver.com/rooms?ms=(위도,경도),16&a=(검색옵션)=ONEROOM

나는 모든 유형의 원룸 매물을 알고 싶기 때문에 검색옵션은 동일하게 두고(전체선택), 위도/경도만 리스트에 넣어서 바꿔가며 페이지를 로드하여 정보를 읽게 하였다.

검색옵션(공통): APT:OPST:ABYG:OBYG:GM:OR:VL:DDDGG:JWJT:SGJT:HOJT&e=RETAIL&aa=SMALLSPCRENT&ae



**Q: 왜 ④의 버튼을 누르는 방식을 쓰지 않았는가?**

A: 버튼을 정확하게 눌렀음에도 불구하고 페이지가 엉뚱하게 로드되는 현상이 몇몇 지역에서 발견됨. 그래서 반복문을 돌리면 오류가 나거나 잘못된 정보를 가져오게 되어 부득이하게 계속 페이지를 새로 로딩하는 방식을 사용함.

예시)서울시 영등포구 양평동을 클릭했을 때 양천구 목동 정보가 뜸

이럴 경우 다시 영등포구 → 양평동을 선택해줘야 바른 화면이 뜸. 하지만 그것도 일시적이라 다른 지역을 좀 클릭하다 다시 양평동 검색시 목동 정보가 로드됨.

![image](https://user-images.githubusercontent.com/33304926/147572694-febb1109-0c99-4855-9cdc-dbd7e2162017.png)



**Q: API를 찾아서 가져오는 방식이 빠르지 않을까?**

A: API를 찾았으나, Postman에서는 제대로 데이터를 가져왔지만 python으로 실행시켰을 때는 엉뚱한 정보를 가져옴. 원인을 알 수가 없어 웹드라이버를 사용함.

예시)포스트맨 요청 결과

![image](https://user-images.githubusercontent.com/33304926/147572772-0551a2ad-2c78-47f9-943e-7ad53416c78d.png)

![image](https://user-images.githubusercontent.com/33304926/147572811-89c57628-b83c-4c17-b6ac-271b62a63240.png)

파이썬 요청 결과

![image](https://user-images.githubusercontent.com/33304926/147572844-fa33383b-7417-43bb-bb4e-1bf22c154e10.png)

![image](https://user-images.githubusercontent.com/33304926/147572868-87d2c435-6e20-4b28-aa47-4a731ab91218.png)

또한 공공기관의 데이터(https://github.com/vuski/admdongkor)는 경계면(변두리)의 위도/경도를 표시해 주기 때문에, 지역의 중심에 있는 위도/경도가 필요한 내가 사용하기에는 어려움이 있었음.



## 크롤러 구조

내가 만든 크롤러(Crawl_final.py)의 구조는 크게 5부분으로 나뉜다.

1. 필요한 패키지들을 import

   ![image](https://user-images.githubusercontent.com/33304926/147573236-e9cdc27d-34d8-42c8-b765-2332ce43a327.png)

2. 서울, 경기도, 인천의 각 동네 (위도, 경도) 정보

   ![image](https://user-images.githubusercontent.com/33304926/147573336-5f138ab8-6e78-4cf6-a1da-38572d269e17.png)![image](https://user-images.githubusercontent.com/33304926/147573414-be7ab25d-4056-4390-b2b3-a47a6028a99c.png)![image](https://user-images.githubusercontent.com/33304926/147573431-bcc0cd09-31d8-469d-a58c-a030ba1c32e1.png)

   

3. 검색 옵션 부여

   ![image](https://user-images.githubusercontent.com/33304926/147573502-5646caba-f49a-4b27-9bba-cc1352cf9e1e.png)

   

4. 스크롤 내리기

   ![image](https://user-images.githubusercontent.com/33304926/147573552-67dc7e5b-65cf-4625-a352-b47abf77dc9b.png)

   - 사이트 왼쪽의 매물 리스트를 무한스크롤 끝까지 내려줌(loader라는 div가 없을때까지 내려주면 됨)
   - 다 내린 뒤 무한 스크롤에 의해 새로 등장한 데이터들을 업데이트하기 위해 BeautifulSoup로 페이지를 다시 한번 읽어줌

   

5. 페이지 정보 가져오기

![image](https://user-images.githubusercontent.com/33304926/147573697-5c5caf02-5e6f-4160-a06d-7a60612fbaa2.png)

![image](https://user-images.githubusercontent.com/33304926/147573733-af2b42ce-6faf-4cdc-b57b-47636e87151a.png)

- 현재 페이지에서 각 영역에 해당하는 css selector의 item들을 모두 골라 항목별로 배열에 넣고, 각 배열에서 1~n번째 아이템들을 뽑아서 하나의 list로 만든다. 그리고 이걸 total_result라는 list에 넣는다. 이때, 도시,구,동은 고정으로 넣는다.

![image](https://user-images.githubusercontent.com/33304926/147573896-059cf80f-5aa1-4537-ae9e-ff44733dab66.png)

![image](https://user-images.githubusercontent.com/33304926/147573919-a2fe8ee8-0055-4e0d-b459-b1b0ef4f1fe6.png)

- 데이터를 dataframe으로 바꿔준 뒤, 중복되는 행을 제거하고 csv파일에 누적 저장한다.





## 데이터 클렌징

분석을 위해 수집한 데이터를 한번 정제해 주는 작업이 필요하다.

![image](https://user-images.githubusercontent.com/33304926/147574019-94d3b259-ab15-4b4a-8ae7-879ae10ff72f.png)

- 저장해놓은 데이터를 불러온 뒤, 필요없는 열을 제거하고, 중복되는 행들을 삭제한다.



#### 공급면적

![image](https://user-images.githubusercontent.com/33304926/147574060-6425a30a-6cd1-4712-9e36-e4ab1d7c27bb.png)

- / 앞의 숫자가 공급면적을 나타낸다. 숫자 옆의 알파벳, 한글은 집의 유형을 의미한다.

  ex) 같은 아파트라도 25평 A형, 25평 B형이 있음

- 면적 당 가격 계산을 위해 공급면적만 따로 빼준다(면적 당 계산은 공급면적이 기준).
- 숫자 옆의 알파벳, 한글은 집의 유형을 의미하므로 제거해줘도 면적 당 가격 계산에 상관이 없다.
- 이제 etc column은 쓸모가 없으므로 삭제해준다.



#### 가격

![image](https://user-images.githubusercontent.com/33304926/147574226-d883a2ce-7d73-4f66-b6db-7ba1d7153bce.png)

- 가격은 총 3가지의 유형이 있다.
  1. 보증금이 있을 경우, (보증금/월세) 형식으로 가격이 표시되므로 가격에 /이 들어간다. 따라서 가장 처음 나오는 /앞의 값을 보증금으로 가져올 것이고, /뒤의 수는 월세로 가져올 것이다. 
  2. 전세는 /가 들어있지 않은 것들을 가져오면 된다.
  3. 세번째가 바로 중간에 가격이 변동된 경우인데, 이 경우 데이터를 분리하는것이 어려우니 데이터 분석에서 제외할 것이다.



#### 보증금

![image](https://user-images.githubusercontent.com/33304926/147574458-ee53c50c-9d62-4aef-a700-7c94a98a106e.png)

- 보증금 가격의 유형은 총 3가지로 나뉜다.
  1. a억: 이 경우, a을 숫자로 바꾼 다음 a * 100000000을 수행해주면 숫자로 값을 치환가능하다.
  2. a억 b천만원: 이 경우, a을 숫자로 바꾼 다음 a * 100000000 해주고, b을 숫자로 바꾼 다음 b * 10000해준 뒤 두 값을 더해주면 숫자로 값을 치환가능하다.
  3. b천만원: 이 경우, b을 숫자로 바꾼 다음 b * 10000을 계산하면 숫자로 값을 치환가능하다.
  4. 나머지는 전세를 의미하므로 0을 채워둔다.



#### 월세

![image](https://user-images.githubusercontent.com/33304926/147574688-ffbded3f-50af-4551-92c8-b08b3c2ed39b.png)

- 월세 정보의 경우, 앞서 보았던 중간에 가격이 변동된 경우에 의해 잘못된 데이터가 있을 수 있다. 그걸 걸러내기 위해 몇 가지 방법을 사용하였다.
  1. 먼저 월세정보에 억이 있을 경우, 해당 행을 제거한다.
  2. 월세 정보에 억이 없더라도, 길이가 3자리를 넘어갈 경우 해당 행을 제거한다.
  3. 위에 두 경우에 모두 해당되지 않는다면 비로소 해당 숫자를 int로 만들고 10000을 곱한다.
  4. 나머지의 경우, 전세를 의미하므로 0을 넣어준다.



#### 전세

![image](https://user-images.githubusercontent.com/33304926/147574854-d4833c1f-156b-4596-bd8a-b137358c389d.png)

- 억을 기준으로 데이터를 나눠보면, 전세 정보의 유형도 여러가지로 나뉜다.
  1. a억:
     - a억일 경우: 이 경우, a을 숫자로 바꾼 다음 a * 100000000을 수행해주면 숫자로 값을 치환가능하다.
     - n 중간에 가격이 변동된 경우①(1n억 -> 1m억으로 바뀐경우 1n1m억으로 가격이 표현됨): 길이가 4 이상이면 해당 행을 삭제한다.
  2. a억 b천만원:
     - 중간에 가격이 변동된 경우②(1억x천 -> 1억z천으로 바뀐경우, 억을 기준으로 split했을 때 x1억 으로 표현됨. 무엇보다 코드에서 b는 a억 b천에서 b를 의미하기 때문에 b에 억이 들어가선 안 됨): b에 억이 있을 경우 해당 행을 삭제한다.
     - 중간에 가격이 변동된 경우③((1억x000 -> z000)으로 바뀐경우, 억을 기준으로 split했을 때 x000z000으로 표현됨.): b의 길이가 5를 넘을 경우 해당 행을 삭제한다.
     - a억 b만원일 경우: a * 100000000를 수행해주고, b * 10000를 수행해준 뒤 더해준다.
  3. b천만원
     - 중간에 가격이 변동된 경우④(x000 -> z000으로 바뀐경우, x000z000 으로 표현됨): 길이가 4를 넘을경우, 해당 행을 삭제한다.
     - b천만원일경우: b * 10000를 수행해준다.
- 나머지의 경우 월세/단기임대를 의미하므로 0을 넣어준다.
- 이제 ‘Price’ column은 필요 없으므로 삭제함



![image](https://user-images.githubusercontent.com/33304926/147575367-c50c5e17-6328-496d-be4e-7df64c4c56df.png)

- zeppelin에서 분석을 수행하기 위해 한글을 영어로 고쳐준다(한글이면 오류가 나는 경우가 많아서 영어로 바꿔줌).
- 중복제거를 한번 더 해준 뒤, testClean_3_6.csv에 데이터를 저장한다.





## 데이터 분석

Zeppelin 사용을 위해 putty와 Cyberduck을 이용해 데이터를 hdfs에 업로드한다.

![image](https://user-images.githubusercontent.com/33304926/147575526-1f8d287b-286f-4f1b-bf30-5847952e1081.png)

Spark를 이용해 hdfs에서 정제한 데이터(TestCrawl_3_6.csv)를 읽어와 데이터프레임으로 만든다.

![image](https://user-images.githubusercontent.com/33304926/147575560-74dcf986-66bd-4894-81aa-0848397c30b5.png)

위에서 읽은 데이터를 임시 테이블(houses)로 만든다. 



이후 Zeppelin으로 몇가지 분석을 진행하였다.

**Q**: 서울, 인천, 경기도에서 **가장 원룸 물량이 많은** 도시는?

![image](https://user-images.githubusercontent.com/33304926/147575660-5215453c-fb34-4d1b-bd48-c23b1fb2f63c.png)

![image](https://user-images.githubusercontent.com/33304926/147575691-6bcfa8dd-f3e9-4242-8766-b2cc02cd2e80.png)

**A**: **서울**이 **약 8만 4천여건**으로 가장 많았고, 그 다음이 경기도(약 3만), 그 다음이 인천(약 7천)이었다.



**Q**: 보증금/월세/전세의 **평균 가격**이 가장 높은 10개의 구는?

![image](https://user-images.githubusercontent.com/33304926/147575843-8a26199b-8288-48ce-97f5-50f432245238.png)

![image](https://user-images.githubusercontent.com/33304926/147575857-37162301-5a93-4e57-bcae-4290c6d6f0d2.png)

**A**:

**보증금**: 과천시, 종로구, 용산구, 송파구, 서초구, 성남시 분당구, 성동구, 마포구, 강남구, 서대문구

**월세**: 종로구, 강남구, 과천시, 용산구, 서초구, 중구, 성남시 분당구, 송파구, 마포구, 성동구

**전세**: 용산구, 강동구, 송파구, 영등포구, 성남시 수정구, 성동구, 서초구, 하남시, 강남구, 중구

보증금, 월세, 전세 모두 8개지역이 **서울**, 2개지역이 **경기도**였다.



**Q**: 전세, 보증금, 월세의 **실 거래가**가 높은 도시는?

![image](https://user-images.githubusercontent.com/33304926/147576018-65fc8535-05c4-4b38-948d-bb3aac813ecb.png)

![image](https://user-images.githubusercontent.com/33304926/147576027-906f50db-72a6-409b-89a5-f07ee077640f.png)

**A**: **서울**이 모두 가장 높았다. 경기도랑 인천의 경우, 보증금 차이는 그렇게 크지 않았다.



**Q**: 평균 전세/보증금/월세의 **실 거래 가격**이 높은 10개의 지역은?

![image](https://user-images.githubusercontent.com/33304926/147576174-684f0f14-cbc9-4dd7-b59a-d6a685e22ce9.png)

![image](https://user-images.githubusercontent.com/33304926/147576185-5f0e2d78-1045-458c-96cd-b45459596095.png)

**A**: **서울**이 모두 가장 높았지만, 보증금/월세 가격은 **경기도**도 꽤 높았다. 

**전세**: 양천구, 영등포구, 증랑구, 성남시 분당구, 용산구, 강남구, 은평구, 서초구, 성동구, 동작구

**보증금**: 종로구, 고양시 일산서구, 용산구, 강남구, 성남시 분당구, 서초구, 송파구, 수원시 영통구, 파주시, 서대문구

**월세**: 종로구, 용산구, 안양시 동안구, 중구, 용인시 수지구, 수원시 영통구, 서초구, 강화군, 이천시, 강남구



**Q**: **평(약 3제곱미터)당 가격**이 높은/낮은 30개의 지역은(전세 기준)?

![image](https://user-images.githubusercontent.com/33304926/147576315-11e8bce4-4816-410f-aaca-6bdf5af59284.png)

![image](https://user-images.githubusercontent.com/33304926/147576354-c7575f61-a70b-4924-bdd3-fd65fe617599.png)

**A**: **높은순**으로는 **서울**이 23지역, 경기도가 7지역이었고, **낮은순**으로는 **경기도**가 22지역, 인천이 8지역이었다. 





## 결론 및 발전가능성

수도권에서도 서울이 가장 물량이 많았고, 가격이 평균적으로 높았다. 하지만 실 거래가로 비교해보면 보증금과 월세는 경기도에서도 비싼 지역이 있었으며, 전세의 경우 인천과 경기도의 가격 차이가 생각보다 크지 않았다. 경기도의 경우 평당 가격이 높은 30개의 지역과 낮은 30개의 지역에 모두 들어가 있는 것으로 보아 지역별로 편차가 심한 것으로 보인다. 

이 데이터는 부동산 정보를 원하는 사람들, 그리고 기존에 부동산 관련 사업을 하는 사람들에게 큰 도움을 줄 것으로 사료된다. 예를 들어, 다방/직방의 경우 부족한 실거래가 데이터를 제공해 줄 수 있을 것이다. 소비자의 경우, 실제 거래가 이루어진 가격과 판매자가 올린 가격을 비교하면서 합리적인 소비가 가능해질 것이다. 

또한 나는 능력의 한계로 이루지 못했지만, 지역별 평당 가격을 선형 회귀 분석을 이용해 예측해볼 수도 있을 것이다. 이 경우, 시세 파악은 물론 미래의 원룸 거래를 미리 파악하여 이익을 얻는 것이 가능할 것이다. 





## 참고문헌

[실습]Selenium스크래핑(3학년 1학기 인공지능 자료)

네이버 부동산 크롤링(전세)https://www.youtube.com/watch?v=-LPHw_xPyE4&t=236s

네이버 부동산 크롤링(구버전, 모바일버전) https://blog.naver.com/PostView.nhn?blogId=inasie&logNo=221353956889