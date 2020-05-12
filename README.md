# 강의 영상 크롤링하기

# 시작하기

코로나19로 인하여 거의 대부분의 수업이 녹화 강의로 바뀌었습니다. 우리는 매 수업 때마다 이클래스에 들어가서 강의를 하나씩 보고 수강 기록을 남겨야 합니다.

그런데 말입니다. 불편하단 말이죠.

강의 들을 때마다 3초씩 학교 마크 나오는 것도 싫고 불필요한 내용을 건너뛰게도 못하게 해 놓았습니다. 특히 매번 로그인하는 건 정말 귀찮아요

자 그럼 어떻게 하는가?를 고민해 보면 저는 동영상을 모조리 다운 받아 버리자는 획기적인 결론에 이르렀고, Selenium을 이용해 목적을 완수할 작당을 꾸밉니다.

# 1. 구성하기

크롤링 기술은 당연히 마법의 Selenium을 사용해야겠죠. 그리고 프로그래밍 언어는... 오랜만에 다뤄보는 파이썬으로 만들어보죠! 파이썬이 다른 언어보다 기본 문법이 간단하고 파이프라인을 만들기 편해서 보기가 좋거든요. ~~다른 언어는 main 만들어야해서 지저분함...~~

또한 크롬 브라우저를 이용해서 작전을 펼쳐보아요. 크롬 개발자 도구가 기능이 많고 직관적이라서 사용하기 편리하답니다.

# 2. 작당하기

## 1. 로그인 뿌셔

먼저 로그인을 해야합니다. 저는 로그인하지도 않고 강의들의 고유값을 따오는 방법을 모르거든요 ㅠㅠ ~~아시는 분 연락 좀~~ 로그인하는 건 구글에 Selenium 검색하면 수두룩하게 나오니까 패스~

아 로그인 너무 자주하니까 서버에서 이상징후 감지해서 차단 당했어요😥 그 다음부터는 왠지는 모르겠지만 자동 로그인 기능이 안먹히네요. 계속 오류가 남

## 2. 수강 목록 가져오기

로그인을 하고 대시보드를 들어가면 내가 현재 듣고 있는 수업이 카드 형식으로 떠 있어요. 이제 저것들을 모두 분해해서 강의 이름과 강의 고유번호를 추출해야 합니다.

이 부분도 HTML 자세히 보고 반복문 돌려서 하나씩 배열에 넣으면 쉽게 끝난답니다!

다만 배열에 넣을 때 자료형을 어떻게 구성할지는 좀 더 고민해 봐야겠어요. 지금은 단순히 courseName, courseNumber 배열에다가 넣었는데 클래스로 짜야할까 싶어요.

## 3. 각 강의별로 영상번호 추출하기

여기가 제 코드의 가장 큰 고민거리이자 분기점이에요. HTML 분석을 계속하다 보니 각 영상에는 ID값이 할당 되어 있더라고요. 그 아이디 값만 알면 굳이 로그인하지 않아도 영상을 추출할 수 있다는 사실을 알게 되었어요

이 사실을 알고 나서 코드의 진행을 결정을 했는데 "영상 번호만 죽 추출한 다음 다운을 받자"였죠. Selenium의 장점이자 단점이 현실 그대로 표현하는 건데 그게 서버에서 HTML파일을 일일이 기다려야 하니 속이 터져요. 

결정을 하고 추출하려 하니까 의외의 벽에 막혔어요. 바로 iframe 안의 HTML소스가 안 보이는 거에요.

### Iframe 처리하기

Selenium을 다루시는 분들을 머릿속에 박아놔야 하는 내용이라 생각해요

크롬 개발자 도구에서는 Iframe 안이 보일 건데 파이썬에서는 계속 안보여요. 그 이유가 iframe은 부모 HTML이 자식 HTML을 표현하는 수단이라서 그런가봐요. 그럴 때는 Selenium에서 지원하는 switch_to.frame을 사용하셔야 되요.

```python
# iframe에 접근하기
iframe = driver.find_element_by_id("id")
driver.switch_to.frame(iframe)

# 부모 HTML로 돌아가기
driver.switch_to.parent_frame()
```

### 클릭이 안될 때

이제 각 강의를 클릭하신 후 영상 정보를 가져 오면 됩니다. 이때도 기억해야 할 것 하나!

Div는 버튼이 아니라서 자동으로 클릭이 안돼요ㅠㅠ 이럴 때에는 강제로 javascript 단에서 이벤트를 호출하면 스무스하게 성공합니다.

```python
div = driver.find_element_by_id("id")
driver.execute_script("arguments[0].click()"), div)
```

## 4. 영상 다운받기

영상 다운 받는 것은 쉬위요.

그래서 스레드를 만들어서 다운 받아 보는 것을 시도해 보고 OS 시간에 배운 Semaphore도 써먹어 보았습니다.

아 그리고 write 명령의 반환값이 1238이면 비정상적인 동영상을 다운 받는 거더라구요

```python
import requests
import os
import threading, requests, time

def createFolder(directory):
    try:
        if not os.path.exists(directory): os.makedirs(directory)
    except OSError:
        print ('Create '+ directory +' Folder Error')

def download(className, root_dir, numSection, video_id, video_name):
    createFolder(root_dir + "/" + className)
    createFolder(root_dir + "/" + className + "/" + str(numSection+1)+"주차")
    
    if len(video_name) > 10: video_name = video_name[:10]
    
		# contents부분의 url이 불분명해 그냥 무조건 때려 넣어보는 방법;;
    for idx in range(10):
        try:
            url = "https://cau.commonscdn.com/contents"+str(idx)+"/cau1000001/"+video_id+"/contents/media_files/mobile/ssmovie.mp4"
            myfile = requests.get(url)
            code = open(root_dir+"/"+className+"/"+str(numSection+1)+"주차"+"/"+video_name+".mp4", 'wb').write(myfile.content)
            
						# 다운 성공!
            if code != 1238: break       
        except:
            print(code, url)
    if idx == 99:
        print("Download Error at " + className +", week"+ str(numSection+1) +", code: "+ video_id)
        
def download_all(vidoes_class, root_dir, classNames):
    sema = threading.Semaphore(10)
    
    for sections, className in zip(videos_class, classNames):
        for numSection, section in enumerate(sections):
            for lecture in section:
                sema.acquire()
                time.sleep(1)
                download(className, root_dir, numSection, lecture[0], lecture[1])
                sema.release()
```

# 3. 결론

확실히 이번 프로젝트를 하면서 Selenium을 더 깊이 배웠다. 특히 iframe에 접근하는 것과 버튼 클릭을 Javascript로 처리하는 것은 꽤나 유용하다.