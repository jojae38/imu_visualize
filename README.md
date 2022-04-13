# IMU_sensor_slip_measure

Sensor_imu와 같이 실행해야 합니다. ROBOT_pose (orientation) 와 IMU_sensor(orientation) callback 이 모두 들어오는 시점에서 실행됩니다.
처음 코드를 실행 시 로봇을 움직이지 말고 2초간 놔 두시길 권장합니다.

1. callback 이 모두 들어온 경우 1초간 양쪽의 orientation 값을 이용해 차를 구한 후 해당 값을 0으로 맞춰줍니다. 예) 로봇 각도 1.3 IMU 각도 2.6 차 = 1.3
차이 1.3제거

![image](https://user-images.githubusercontent.com/58541374/160034868-fa3bb3fe-fa37-4f09-8b36-ed8616f15fd5.png)

IMU와 로봇 간의 각도 차이가 1.943정도 나오기에 제거해주는 모습

2. geometry_msgs::Twist 를 이용해 **/acc_diff** 토픽명으로 publish 를 진행하며 다음과 같이 나타냅니다.
* [angular.x = dθ] 
* [angular.y = dθ'] 
* [angular.z = dθ"] 
3. rqt_plot을 실행 후 y 축을 적절히 조정하면 로봇과 IMU 센서 간의 각도차이 및 각도 변화량이 측정 가능합니다.
![image](https://user-images.githubusercontent.com/58541374/160034918-8005b8ba-b8f8-4acd-b125-33e26b54124e.png)
4. **/robot_adjust_byslip** 에 의해 일어난 슬립의 정도를 Quaternion으로 publish 하게 된다. 추후 로봇과 연결해서 pose가 publish 된 Quaternion 을 적용하여 수정 가능한지 알아낼 **(예정)**

## 주의사항

* Sensor_imu도 15Hz로 로봇과 통일하였다 1/15초 변화량을 생각하면 dθ에서 dθ'로 변환 시 30을 곱해야 하지만 값이 너무 크게 변하여 관측이 어렵기에 곱을 하지 않았습니다.
* launch 파일 내 dθ" 변화량이 얼마정도 나오면 슬립으로 인식할 것인지 슬립 민감도를 설정하는 파라미터가 있습니다. -> 현재 0.1
* 현재 IMU drift 현상 (가만히 놔 둬도 각도가 찔끔찔끔 변하는 현상) 이 10초에 (+-) 0.0002~0.0003 정도 관측되고 있습니다.
* 초기값을 빼기 원하지 않으시는 경우에는 launch 파일 내부에 notinit 항목에 0을 넣어주시면 됩니다.

![image](https://user-images.githubusercontent.com/58541374/160035016-56a5da68-631f-49f8-87fa-3455789f9c81.png)

움직이지 않았는데 20초 후 변화한 각도

![image](https://user-images.githubusercontent.com/58541374/160035045-2b010514-8ba8-4992-8e84-7172cca5cd06.png)

 * IMU와 로봇의 각도 정확도

![image](https://user-images.githubusercontent.com/58541374/162864692-f9a2fd35-83e7-43f0-817e-90ff6f78e711.png)



## 개선사항

<br>(Issue1) - ***이중 미분 값이 올라갔다 내려갈때를 슬립이 일어난 지점으로 가정하고 슬립 일어난 지점 전 후의 구간을 측정하여 IMU센서 값을 기준으로 로봇의 각도를 보정해줘야 한다.(완료) ->현재 이차 미분값의 변화량을 확인하여 수정해야할 값을 publish 함***
<br>(Issue2) - **drift** 현상 보정
<br>(Issue3) - **IMU**의 틀어진 각도를 다시 보완하는 기능
![image](https://user-images.githubusercontent.com/58541374/161917032-5e5ab851-108d-492c-8527-59d35097d363.png)

<br>(Issue4) - 어차피 **orientation 차이**를 yaw로 바꾸는 것이라 로봇이랑 imu가 동시에 한바퀴를 회전해도 상관은 없을테지만 초기 각도 차이가 클 경우에는 3.14에서 -3.14로 확 바뀔 염려가 있음
예) dθ=3.1 여기에 0.04만 차이가 더해질 시 -3.14로 변한다. 고로 드리프트가 발생했다고 오판할 가능성이 있다. 전 각도차이 -3.12 현 각도차이 3.12 => 실제론 dθ=0.04의 각도가 변했지만 각도차의 변화량 입장에선 dθ=6.2가 변했다고 인식함
<br>(Issue5) - IMU 고정문제가 존재 만약 uart통신이 가능하다면 uart를 시리얼로 바꿔주는 모듈도 필요가 없고 나중에 IMU를 통합보드에 붙여서 사용할 수 있게 됨 (Sensor_IMU_uart 참조) (진행중)
<br>(Issue6) - 현재 민감도를 0.2로 설정하였다.(민감도를 초과 시 slip이 일어났다고 가정한다.) slip시 관측되는 ddQ 값은 약 0.25 임의로 20도 정도 로봇을 들고 돌렸을 경우이다.
<br>하지만 조이스틱으로 로봇을 조작할 경우 0.3 의 민감도를 초과하는 ddQ 값이 관측된다. 즉 평범하게 회전하는게 값 변화율이 더 크다.

## 시도

(Issue4 의 대응) -> 아예 현재 dθ 와 과거(1/rate 초 전) dθ 의 Quaternion 차를 구해서 미분값을 구한 후(Quaternion) 이걸 yaw로 바꿔주고
<br>동일하게 이중미분도 현재 dθ' 와 과거(1/rate 초 전) dθ' 의 Quaternion 차를 구해 이중미분 값을 구한 후(Quaternion)으로 바꿔 이걸 yaw로 바꿔줄 생각이였으나
<br>(이 경우 대회전을 하지 않는 이상 미분값이 극과 극으로 뛰는 현상은 방지할 수 있다. -3.14 -> 3.14 => 6.28*30 = 188.4)
<br>너무 작은 Quaternion값을 예) x: 0.0010 y:0 z:0 w:0 yaw로 바꿔줄 경우 -nan값이 관측됨 사용 불가

(Issue4 의 대응) -> Quaternion 설명을 자세히 읽어봤어야 했다. x, y, z, w 모든 값이 sin 값처럼 움직인다. 범위(1~-1)
<br>**즉 내가 서로 빼거나 더한다고 해도 orientation 이 일정한 값이 나오지 않는다.(위치에 따른 각도 변화에 따라 z,w가 다르게 변할 수 있기 때문이다.)**  
그래서 가끔 차나 합이 2에 가까운 값이 나오기도 하는데 이는 당연히 틀린 값일 수 밖에 없다. -> 결국 각각의 orientaion 차를 yaw로 바꿔주면 안되고 각각의 orientation을 yaw로 바꿔준 후 360도에서 조건문으로 조정을 해줘야 한다.

(Issue4 의 대응) -> 위 그림 **IMU와 로봇의 각도 정확도** 에서 봤듯이 어차피 로봇과 IMU센서간 각도 차이가 너무 크다. 
<br>고로 각도 차이가 정확할 필요는 없기에 PI 에서 -PI 로 변할 때 혹은 -PI 에서 PI 로 변할 때 보상값인 2PI 를 더해주거나 빼 주었다. 
<br>이제 왼쪽으로 계속 회전하면 각도는 계속 올라갈 것이고 오른쪽으로 회전시 각도는 계속 내려갈 것이다. 어차피 쌍미분 차이가 크게 나타난 곳을 슬립이라 가정하고 그 시간차이를 구해서 그 시간차이 동안의 IMU의 quaternion 값의 차를 구해서 보상값으로 보내주면 될 것이라 예상한다.

(Issue6 의 대응) -> 로봇의 회전 속도를 낮춰본다 개선 여부 측정
<br> 결과적으로 불가 25에서도 느린데 5정도는 되어야 회전시 슬립이 아니라고 측정됨 근본적인 해결 방법은 아님

(Issue6 의 대응) -> 로봇을 한바퀴 여러번 돌려본 결과 Imu는 2PI에 가깝게 나오지만 Robot은 더 높게 나오니 특정 수를 곱해줘서 보정을 해주자.
<br> IMU 로봇 둘다 한 바퀴를 회전시 왼쪽으로 회전하나 오른쪽으로 회전하나 IMU는 6.28-6.24정도의 차이를 보였으며 ROBOT의 경우는 10.08-9.98 정도의 차이를 보였다.
<br> 0.01 단위는 컨트롤 이슈라고 가정한다면 **로봇은 6.28/10 = 0.628 의 수치**로 각도 보정이 필요하다. (로봇 rotation 속도 25)
<br> 보정 결과 아무리 회전해도 각도 차이의 차이가 잘 발생하지 않음 (성공)

(Issue6 의 대응) -> 뭔가 이상하다 조이스틱으로 회전시 간혈적으로 슬립이라 판단한다 그에 비해 각도 차이는 변함이 없다.
<br>하지만 외력으로 회전시 슬립이라 판단 안할 때가 있으며 각도 차이가 변한다. 
<br>(현재 1/15초 짧은 시간 변화의 이중 가속도로 슬립을 판단중이다. 이걸 1/5 혹은 1/3 혹은 1초까지 늘려서 판단하는 시스템을 만드는 것도 나쁘지 않을 것 같다.)

(Issue6 의 대응) -> 이건 예상이긴 하지만 Robot의 데이터 값과 IMU의 데이터 값의 미묘한 시간 차이 때문에 dQ 값 차이가 발생하는 것 같다.
<br>예를들면 회전중에 다음과 같은 3가지 case가 겹쳐서 dq가 발진을 하는 것이다.
<br>Case 1: IMU값은 업데이트 되었지만 로봇 값은 업데이트 되지 않은 경우 
<br>Case 2: 로봇은 업데이트 되었지만 IMU는 업데이트 되지  경우 
<br>Case 3: 모두 업데이트 된 경우
<br>->결과 큰 차이가 없다. 근본적인 식 조정이 필요할 것 같다.
