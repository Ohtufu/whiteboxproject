## white box project

### 1.시스템

* 시스템 구성도  
![image](https://user-images.githubusercontent.com/55979143/79418570-f1458d80-7fef-11ea-997a-bc6f0a8a0719.png)
위의 구성도와 같이 스쿨버스에 카메라 역할을 하는 라즈베리 파이 Zero kit 2개를 설치합니다.
또한 App을 통해 외, 내부를 실시간 스트리밍 시청 가능하며 좌석 매핑에 대한 좌표를 설정, 저장해서 OS와 서버 담당인 라즈베리 파이 B3에서 송수신합니다. 운행 종류 후 '종료' 버튼을 누르면 ROI, Histogram 이용한 영상처리로 버스 내부의 아이 유무를 판단합니다.  

* 시스템 흐름도   
![image](https://user-images.githubusercontent.com/55979143/79416387-880f4b80-7fea-11ea-99f3-404e03fedeb0.png)  
구체적인 흐름도입니다.

### 2. 서버(Raspberry pi B3:python)  
![image](https://user-images.githubusercontent.com/55979143/79416328-58f8da00-7fea-11ea-96bc-41e7e5cdf1c9.png)  

#### 1)Homography  

Homography 기법은 3D의 이미지를 2D처럼 평면화를 해주기 위해 사용했습니다.
좌석을 매핑할 때 카메라의 위, 옆에서 찍히는 좌석은 대각선으로 찍히게 됩니다. 성인과는 다르게 아이의 경우에는 좌석에서 차지하는 비율이 상대적으로 더 적은데 대각선으로 찍히게 되면 그 비율마저 더 작아지게 되므로 이미지 비교 전에 평면화를 통해 더 확실한 탐지를 합니다.  
![ho](https://user-images.githubusercontent.com/60215726/74673767-408b5f80-51f3-11ea-9063-1f3d91e6b167.PNG)
왼쪽 이미지는 원본 이미지입니다. 중앙의 이미지는 ROI만 적용한 경우이며 마지막 이미지는 Homography+ROI를 적용한 이미지입니다.
OpenCV에서 perspective transformation = homography 관계이며, cv2.getPerspectiveTransform( )와 cv2.findHomography( )로 perspective 변환과 homography를 각각 지원하는데 4개의 점만을 이용하여 변환 행렬을 찾는 cv2.getPerspectiveTransform( )을 이용하였습니다.
변환 행렬을 구하기 위해서 cv2.getPerspectiveTransfom() 함수를 이용하고 cv2.warpPerspective() 함수에 변환 행렬 값을 적용하여 최종 결과 이미지를 얻는 것입니다.  
```python
#[x,y] 좌표점을 4*2의 행렬로 작성
#좌표점은 좌상 -> 좌하 -> 우상 -> 우하
pts1 = np.float32([list(point_list[0]),list(point_list[1]),list(point_list[2]),list(point_list[3])])
# 좌표의 이동점
pts2 = np.float32([[0,0],[weight,0],[0,height],[weight,height]])

M = cv2.getPerspectiveTransform(pts1,pts2)

img_result = cv2.warpPerspective(img_original, M, (weight,height))
cv2.imshow("img_Homograph", img_result)
```
(단, 프로젝트 당시 코드에서는 모형으로 진행하였고 다른 문제로 인하여 Homography 부분은 제외하고 진행하였습니다. 이 코드 소스는 따로 Homography.py 명으로 업로드되어 있습니다.)

[참조](https://opencv-python.readthedocs.io/en/latest/doc/10.imageTransformation/imageTransformation.html)

#### 2)ROI(Region of Interest)

원본 이미지에서 관심영역을 추출할 수 있도록 해주는 영상처리 기법입니다.
```python
def startROI():
	#서버에서 사용된 코드소스입니다.
	i=0
	img_start=cv2.imread('/home/pi/Desktop/whitebox/startimage.jpg')
	while i<=point.count:
		now='Start_ROI'+str(i)
		subimg_start = img_start[y1_list[i]:y2_list[i], x1_list[i]:x2_list[i]]
		#위의 형태로 img_start[y1_list[i]:y2_list[i], x1_list[i]:x2_list[i]]
		#이 부분에서 ROI를 해주는 것입니다.[처음 y좌표 : 마지막 y좌표, 처음 x좌표:마지막x좌표]로 
		#표시하였고 현재 프로젝트에서는 여러개의 ROI가 필요하여 list형식인 변수로 넣었습니다.
		cv2.imwrite(os.path.join(path,str(now)+'.PNG'),subimg_start)
		#cv2.imshow('start','/home/Start_ROI1.PNG')
		i+=1
```
![hoRO](https://user-images.githubusercontent.com/60215726/74665699-88a28600-51e3-11ea-83bf-0a5b51dccb27.PNG)

#### 3)Histogram  

버스 내부에 아이의 유, 무가 탐지 되었는지를 구별하기 위해 사용하였습니다.
처음 이미지와 비교할 이미지의 각각의 calcHist 명령어를 통해 히스토그램을 계산을 하고 CompareHist 명령어를 통하여 비교한 수치 값과 기준점을 비교하여 아이의 탐지 여부를 파악하였습니다.
![비교](https://user-images.githubusercontent.com/60215726/74666635-35313780-51e5-11ea-8a3b-9bdd5bf3110b.PNG)  
두 이미지의 히스토그램의 수치를 도표로 나타낸 것이며, 아래 이미지는 프로젝트에서 사용한  cv2.HISTCMP_CORREL(상관) 입니다.  

![co](https://user-images.githubusercontent.com/60215726/74666819-8d683980-51e5-11ea-9c94-63ba4956522c.PNG)
```python
def hist_compare(d_count):
	#ROI한 이미지들에서 각각 매핑한 좌석들 중
	#처음 이미지와 비교할 이미지에서 매핑한 좌석만 ROI된 이미지들을 통해
	#두가지로 분류가 된 후 각각 좌표가 동일한 이미지끼리 비교를 하는 부분입니다.
	#즉 Start_ROI0 과 EndROI0이 비교가 되며
	#총 8개의 좌석을 매핑을 했다면 16개의 이미지가 각각 두 개씩 비교가 됩니다.
	#비교가 된 후  cv2.compareHist(H1,H2,cv2.HISTCMP_CORREL)을 통해서
	#값이 나오는데 이 값들은 소수점이 나옵니다.
	#참고 : cv2.compareHist(H1,H2,cv2.HISTCMP_CORREL)에서 나온 값이
	#1에 가까울 수록 유사도가 높은 것이며 1이면 같은 이미지라고 보면 될 것 같습니다.
	#여기서는 값에 10을 곱해 음수부터 10까지에서 6이라는 기준을 잡고
	# 6보다 크면 사람이 없는 것으로 그 이하이면 아이나 사람이 탐지되는 것을
	#구별하였습니다.
	i=0
	msg=''
	while i<=d_count:
		pts1=cv2.imread('/home/pi/Desktop/whitebox/Start_ROI'+str(i)+'.PNG')
		pts2=cv2.imread('/home/pi/Desktop/whitebox/End_ROI'+str(i)+'.PNG')
		H1=cv2.calcHist(images=[pts1],channels=[0],mask=None, histSize=[256],ranges=[0,256])
		cv2.normalize(H1,H1,1,0,cv2.NORM_L1)
		H2=cv2.calcHist(images=[pts2],channels=[0],mask=None,histSize=[256],ranges=[0,256])
		cv2.normalize(H2,H2,1,0,cv2.NORM_L1)
		#현재 아래 부분에서 히스토그램 비교를 하는 부분으로 4가지 비교 중 상관비교를 이용하였습니다.
		d1 = cv2.compareHist(H1,H2,cv2.HISTCMP_CORREL)
		print('df(H1,H2,CORREL)=',d1)
		d1=d1*10
		print(d1)
		print(str(i)+':ROI compare')
		#아래 부분에서 기준점을 나누어 탐지 여부를 정하였습니다.
		if d1>6:
			print('no Human')#
		elif d1<=6:
			print('detech')
			cnum()
		i+=1
	if cnum.count>0:
		print('cnum='+str(cnum.count))
		msg='detech'
		cnum.count=0
	else:
		msg='no people'
	return msg
``` 
[참조 OpenCV:histogram](https://docs.opencv.org/3.4/d6/dc7/group__imgproc__hist.html)

### 3. 카메라(Raspberry pi Zero)
카메라 모듈을 장착한 라즈베리 파이 Zero kit 두 대를 각각 내, 외부를 촬영하는 역할로 사용하였습니다. 내부는 어린아이 유, 무를 확인하는 용도이며 외부는 운전대 기준으로 오른쪽 차량 사이드 미러에 설치하여 아이들의 승, 하차할 때 보이지 않는 사각지대에서의 위험을 예방하기 위해 설치하였습니다.
Wifi 무선 통신을 하여 앱에서 실시간 스트리밍을 위해 실시간 스트리밍 프로토콜(RTSP)를 이용하였습니다.
또한 앱에서와 카메라 자체 내에서 미디어 처리를 위한 Gstramer를 사용했습니다.  

#라즈베리 파이 안에서 RTSP 서버를 열어주고 카메라 스트리밍을 해주는 Bash 스크립트입니다.
**cam.sh**
```linux
#!/bin/bash 
/home/pi/gst-rtsp-server/examples/./test-launch "( rpicamsrc preview=false bitrate=2000000 keyframe-interval=30 ! video/x-h264, framerate=30/1 ! h264parse ! rtph264pay name=pay0 pt=96 )"
```
카메라 역할을 하는 Zero kit들은 따로 모니터가 불필요해서 전원이 켜지면 자동으로 서버를 열고 카메라가 실행이 되도록 설정 을했습니다. 
리눅스 작업 스케줄러인 crontab을 사용하였고 부팅 시 켜지기 위함으로 
**crontab**
```linux
@reboot /home/pi/gst-rtsp-server/examples/cam.sh
```
추가 입력해주었습니다.  
![2020-02-18-173422_1366x768_scrot](https://user-images.githubusercontent.com/60215726/74733387-91e92c80-528f-11ea-969c-58c140736f8a.png)

### 4. APP(Java)
   
#### 1) python과의 소켓통신
소켓 통신을 하기 위한 클래스 부분입니다. 
```java
    //서버
    //서버와 연결하기 위한 연결고리이며 메인 화면에서는 종료버튼에서
    //아이나 사람이 감지되었을 때 종료버튼이 실행되지않고 경고창을 나타내기 위해서 필요합니다.
    public class MyClientTask extends AsyncTask<Void, Void, Void> {
        String dstAddress;
        int dstPort;
        String response = "";
        String myMessage = "";

        //constructor
        MyClientTask(String addr, int port, String message){
            dstAddress = addr;
            dstPort = port;
            myMessage = message;
        }
        //서버와 통신하기위한 부분.
        @Override
        protected Void doInBackground(Void... arg0) {

            Socket socket = null;
            myMessage = myMessage.toString();
            try {
                socket = new Socket(dstAddress, dstPort);
                //송신
                OutputStream out = socket.getOutputStream();
                out.write(myMessage.getBytes());

                //수신
                ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream(1024);
                byte[] buffer = new byte[1024];
                int bytesRead;
                InputStream inputStream = socket.getInputStream();

   /*   notice:
     * inputStream.read() will block if no data return*/

                while ((bytesRead = inputStream.read(buffer)) != -1){
                    byteArrayOutputStream.write(buffer, 0, bytesRead);
                    String msg = byteArrayOutputStream.toString()+"byteArrayOutputStream 이거 실행중";
                    response += byteArrayOutputStream.toString("UTF-8");
                }
                response = "서버의 응답: " + response;
                //Toast.makeText(getApplicationContext(),response,Toast.LENGTH_LONG).show();

            } catch (UnknownHostException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
                response = "UnknownHostException: " + e.toString();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
                response = "IOException: " + e.toString();
            }finally{
                if(socket != null){
                    try {
                        socket.close();
                    } catch (IOException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }
            }
            return null;
        }

        @Override
        protected void onPostExecute(Void result) {
            // recieveText.setText(response);

            ip.setB3_msg(response);
            com_msg=response;
            String comsg = "서버의 응답: detech";
            String comsg2 = "서버의 응답: no people";
            if (comsg.equals(com_msg))
            {      but_num++;}
            else if(comsg2.equals(com_msg)){
                but_num++;
            }
            super.onPostExecute(result);
        }
    }//서버 
```
위의 코드는 생성자 부분만 표시한 코드입니다. 
```java
//constructor
        MyClientTask(String addr, int port, String message){
            dstAddress = addr;
            dstPort = port;
            myMessage = message;
        }
```
MyClientTask의 객체를 생성, 생성자를 변수 순으로 입력하여 사용합니다.
addr은 서버 역할을 하는 ip, port는 8888로 연결되어 있는 동안 변하지 않는 숫자입니다. 초기에 ip를 받아서 변수에 저장하여 사용합니다.  (IpReciver.java 파일에 저장되어 있는 것처럼 카메라와 서버의 Ip를 저장하고 이용할 수 있도록 하였습니다.)
message은 클라이언트가 서버에 명령을 내리기 위한 메시지로 서버에서 저장되어 있는 이름과 동일해야 합니다.
아래의 코드에서 message인 his_msg는 histogram을 나타내고 myClientTask2.execute();의 실행으로 서버에서 histogram인 비교를 시작합니다. 
```java
//java 클라이언트 서버 통신 실행.
static String his_msg = "histogram";
ip_num = ip.getB3_ip();
//아래 두 줄의 코드가 서버에 원하는 행동을 보내는 것으로
//ip_num은 서버에 연결되어있는 wifi의 ip이며, EndROI는 서버에 저장되어 있는 EndROI의 명령어를 사용하기 위해 보내는 것입니다.
//종료버튼에서 처음 명령을 보낼 때 비교할 이미지를 ROI를 하기위해 실행을 해줍니다.
MyClientTask myClientTask2 = new MyClientTask(ip_num, 8888, his_msg);//종료
myClientTask2.execute();
```
위의 코드(java)가 실행하게 되면 아래 서버의 python으로 넘어갑니다.   
```python
#whiteboxserver
#수신한 데이터로 파이를 컨트롤
	res = do_some_stuffs_with_input(data)
	print("pi ac :" + res)
	#클라이언트에게 답을 보냄
	conn.sendall(res.encode("utf-8"))
```   
python에서 수신을 받고 결과물에 따라 답을 보냅니다.   
```java
//클라이언트(java) 클래스 내부에 존재하는 메소드 오버라이딩(histogram비교 후 아이의 유, 무감지 대한 부분)
 @Override
        protected void onPostExecute(Void result) {
            // recieveText.setText(response);

            ip.setB3_msg(response);
            com_msg=response;
            String comsg = "서버의 응답: detech";
            String comsg2 = "서버의 응답: no people";
            if (comsg.equals(com_msg))
            {      but_num++;}
            else if(comsg2.equals(com_msg)){
                but_num++;
            }
            super.onPostExecute(result);
        }
    }
```   
java에서 수신을 한 뒤, response의 변수에 python의 답을 받게 되는 것 입니다.   
   
#### 2) 스트리밍(제로 킷과 앱의 연동)
카메라와 앱에서의 연동을 할 때 이용하는 카메라 IP는 rtsp://192.168.0.0:8554/test 이처럼 구성하며 
안에 있는 192.168.0.0는 라즈베리파이에 연결된 wifi ip입니다.(단, 테블릿과 카메라는 같은 wifi로 연결되어있어야합니다.)     
```java
//SettingActivity.java인 설정 부분
 dlgEdtIP_02 = (EditText) dialogView.findViewById(R.id.dlgEdt2);
 ip.setIp1(dlgEdtIP_02.getText().toString());
```
이처럼 입력받은 ip를 IpReceiver에 저장 후, 사용합니다.

```java
//InsideFullScrren.java 코드
public static String uri2 = ip.getIp2();
```
uri2 변수에 ip를 가져와 사용합니다.
```java
//InsideFullScrren.java 코드
//스트리밍을 하기 위한 공간과 시작.
 VideoView v = (VideoView) findViewById(R.id.videoView04);
 v.setVideoURI(Uri.parse(uri2));
 v.requestFocus();
 v.start();
```	
VideoView를 사용하여 스트리밍을 진행합니다. (추후에는 VideoView 대신에 NDK으로 변경할 예정입니다.)     
   
#### 3) 좌석 매핑
스트리밍 중인 화면을 캡처를 하여 캡처한 이미지에 드래그 터치 이벤트로 좌석을 매핑합니다.
```java
//SeatMapping.java
//불러온 이미지에서 원하는 곳에 좌석을 매핑하는 부분.
//처음에 터치한 곳과 마지막에 터치한 두 점을 서버에 송출.
 int x =0;
            int y =0;

            if (event.getAction() == MotionEvent.ACTION_DOWN) {
                x = (int)event.getX();
                y = (int)event.getY();
                //int형인 x와 y를 ','와 더하여 String으로 만든 이유는
                //서버에 보낼 때 문자열로 인식한 다음 ',' 기준으로 x와y를 나누기 위해 사용하였습니다.
                String msg = x+","+y;
                //처음 터치한 지점에서 서버에 보내는 명령어입니다(아래)
                MyClientTask myClientTask = new MyClientTask(b3_uri, 8888, msg);

                myClientTask.execute();
                tv2.setText(msg);

                // Toast.makeText(getApplicationContext(), "Down", Toast.LENGTH_LONG).show();
            } else if (event.getAction() == MotionEvent.ACTION_MOVE) {

                //  Toast.makeText(getApplicationContext(), "Move", Toast.LENGTH_LONG).show();
            } else if (event.getAction() == MotionEvent.ACTION_UP) {
                int end_x = (int)event.getX();
                int end_y = (int)event.getY();
                String msg = end_x+","+end_y;
                //터치하는 곳을 때는 지점에서 서버에 보내는 명령어입니다(아래)
                MyClientTask myClientTask = new MyClientTask(b3_uri, 8888, msg);
                myClientTask.execute();
                tv3.setText(msg);
                i+=1;
                tv4.setText(i+"번째 좌석지정중");
                // Toast.makeText(getApplicationContext(), "Up", Toast.LENGTH_LONG).show();
            }

            return true;
        }
        else{

            return false;
        }
```
드래그를 진행하는 전체 코드입니다.
   
```java
//터치를 시작하는 코드

            if (event.getAction() == MotionEvent.ACTION_DOWN) {
                x = (int)event.getX();
                y = (int)event.getY();
                //int형인 x와 y를 ','와 더하여 String으로 만든 이유는
                //서버에 보낼 때 문자열로 인식한 다음 ',' 기준으로 x와y를 나누기 위해 사용하였습니다.
                String msg = x+","+y;
                //처음 터치한 지점에서 서버에 보내는 명령어입니다(아래)
                MyClientTask myClientTask = new MyClientTask(b3_uri, 8888, msg);

                myClientTask.execute();
                tv2.setText(msg);
```
먼저 처음 터치를 시작하는 코드에서 좌석에 대한 왼쪽 맨 위를 나타냅니다. x와 y를 event.getX와 Y로 통해 받은 뒤,
String msg에 x+“,”+y를 넣어줍니다. 이 부분에서 x와 y를 ‘,’와 합쳐서 서버에 보내는 이유는 두 번 연속으로 서버에게 통신을 하는 myClientTask.execute()을 한 번으로 줄이기 위함입니다. ‘,’를 붙여준 이유는 x와 y를 구별해 주는 역할로 붙여주었습니다.
   
```java
///터치를 때는 부분코드입니다.
else if (event.getAction() == MotionEvent.ACTION_UP) {
                int end_x = (int)event.getX();
                int end_y = (int)event.getY();
                String msg = end_x+","+end_y;
                //터치하는 곳을 때는 지점에서 서버에 보내는 명령어입니다(아래)
                MyClientTask myClientTask = new MyClientTask(b3_uri, 8888, msg);
                myClientTask.execute();
                tv3.setText(msg);
                i+=1;
                tv4.setText(i+"번째 좌석지정중");
                // Toast.makeText(getApplicationContext(), "Up", Toast.LENGTH_LONG).show();
            }
```
터치를 할 때는 부분으로 오른쪽 아래의 x, y좌표를 서버로 보내주는 코드입니다. (터치를 시작하는 부분과 같고 같습니다.)
여기까지 작업을 완료를 하면 서버에 한 좌석에 대한 매핑이 끝납니다. 
   
#### 4) 종료시
버튼 2번 눌르는 방식으로 진행했었으나 소켓 통신 중에 처음 눌렀을 때의 반응이 없는 것을 확인되었습니다. 그래서 두 번의 클릭 이후 부터 서버의 반응을 확인하고 행동하게한 수정된 방식입니다. 
```java
Button exitButton = (Button) findViewById(R.id.exit);
exitButton.setOnClickListener(new View.OnClickListener(){
            @Override
            public void onClick(View v){
              
                if(but_num==0){
                    AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this);
                    String b3_msg = ip.getB3_msg();
                    String comsg = "서버의 응답: detech";
                    ip_num = ip.getB3_ip();
//아래 두 줄의 코드가 서버에 원하는 행동을 보내는 것으로
//ip_num은 서버에 연결되어있는 wifi의 ip이며, EndROI는 서버에 저장되어 있는 EndROI의   명령어를 사용하기 위해 보내는 것입니다.
//종료버튼에서 처음 명령을 보낼 때 비교할 이미지를 ROI를 하기위해 실행을 해줍니다.
                    MyClientTask myClientTask2 = new MyClientTask(ip_num, 8888, "EndROI");//종료
                    myClientTask2.execute();
                    myClientTask2 = new MyClientTask(ip_num, 8888, his_msg);//종료
                    myClientTask2.execute();
                    Toast.makeText(getApplicationContext(), com_msg, Toast.LENGTH_LONG).show();
                    builder.setMessage("통신중입니다. 다시 한번 눌러주십시오.");
                    builder.setTitle("경고알림창")
                            .setCancelable(false)
                            .setNegativeButton("yes", new DialogInterface.OnClickListener() {
                                @Override
                                public void onClick(DialogInterface dialog, int i) {
                                    dialog.cancel();
                                }
                            });
                    AlertDialog alert = builder.create();
                    alert.setTitle("종료 알림창");
                    alert.show();

                }
                else if (but_num > 0) {
                    AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this);
                    String comsg = "서버의 응답: detech";
                    ip_num = ip.getB3_ip();

                    MyClientTask myClientTask2 =new MyClientTask(ip_num, 8888, his_msg);//종료
                    myClientTask2.execute();
                    Toast.makeText(getApplicationContext(), com_msg, Toast.LENGTH_LONG).show();
                    if (comsg.equals(com_msg)) {
                        builder.setMessage("★★★경고 아이가 남아 있습니다.★★★");
                        builder.setTitle("경고알림창")
                                .setCancelable(false)
                                .setNegativeButton("확인하십시오", new DialogInterface.OnClickListener() {
                                    @Override
                                    public void onClick(DialogInterface dialog, int i) {
                                        dialog.cancel();
                                    }
                                });
                        AlertDialog alert = builder.create();
                        alert.setTitle("종료 알림창");
                        alert.show();
                    } else {
                        builder.setMessage("아이가 없습니다. 종료하시겠습니까?");
                        builder.setTitle("종료 알림창")
                                .setCancelable(false)
                                .setPositiveButton("Yes", new DialogInterface.OnClickListener() {
                                    @Override
                                    public void onClick(DialogInterface dialog, int i) {
                                        finish();
                                        ActivityManager am = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
                                        am.restartPackage(getPackageName());
                                    }
                                })
                                .setNegativeButton("No", new DialogInterface.OnClickListener() {
                                    @Override
                                    public void onClick(DialogInterface dialog, int i) {
                                        dialog.cancel();
                                    }
                                });
                        AlertDialog alert = builder.create();
                        alert.setTitle("종료 알림창");
                        alert.show();

                    }
                    but_num = 0;
                }

            }
        });
```   
종료 버튼을 눌렀을 시, 행동을 결정하는 but_num이 존재하지만 코드에는 없습니다. 같은 클래스 파일에있는 서버에서 응답으로 오는 방식입니다. 앱에서 서버의 비교를 시작하는 명령을 통해 응답의 감지를 뜻하는 detech이 넘어오게 되면 but_num의 변화를 시킨  사용자에게 경고창을 보여주게 됩니다.
```java
//MainActivity.java 에 속하는 서버 메소드 부분이며 but_num을 변화시켜주는 부분입니다.
 @Override
        protected void onPostExecute(Void result) {
            // recieveText.setText(response);

            ip.setB3_msg(response);
            com_msg=response;
            String comsg = "서버의 응답: detech";
            String comsg2 = "서버의 응답: no people";
            if (comsg.equals(com_msg))
            {      but_num++;}
           
            super.onPostExecute(result);
        }
```
          
#### 5) 앱 시연영상
[![Video Label](https://img.youtube.com/vi/j18SoUClJeI/0.jpg)](https://www.youtube.com/watch?v=sx_Ajdshu3k)   
App에서 실행되는 시연영상입니다. 스트리밍과 서버 또한 정상작동 중인 입니다.
   
  
[참조블로그](https://blog.naver.com/cosmosjs/220834751822)
   
### 5. 작품 모형 및 캡스톤 경진대회
![모형](https://user-images.githubusercontent.com/60215726/74937280-1625e600-542f-11ea-8d3b-f7c1deefc52a.PNG)
![경진](https://user-images.githubusercontent.com/60215726/74937275-145c2280-542f-11ea-82e5-ae01d26da9ec.PNG)

![image](https://user-images.githubusercontent.com/55979143/85676665-c8352d80-b701-11ea-9e06-814a1ea177be.png)
