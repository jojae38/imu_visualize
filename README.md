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

#주의사항

* 30HZ이기에 1/30초 변화량을 생각하면 dθ에서 dθ'로 변환 시 30을 곱해야 하지만 값이 너무 크게 변하여 관측이 어렵기에 곱을 하지 않았습니다.
* launch 파일 내 dθ" 변화량이 얼마정도 나오면 슬립으로 인식할 것인지 슬립 민감도를 설정하는 파라미터가 있습니다. -> 현재 0.1
* 현재 IMU drift 현상 (가만히 놔 둬도 각도가 찔끔찔끔 변하는 현상) 이 10초에 (+-) 0.0002~0.0003 정도 관측되고 있습니다.
* 초기값을 빼기 원하지 않으시는 경우에는 launch 파일 내부에 notinit 항목에 0을 넣어주시면 됩니다.

![image](https://user-images.githubusercontent.com/58541374/160035016-56a5da68-631f-49f8-87fa-3455789f9c81.png)

움직이지 않았는데 20초 후 변화한 각도

![image](https://user-images.githubusercontent.com/58541374/160035045-2b010514-8ba8-4992-8e84-7172cca5cd06.png)

#개선사항

<br>(Issue1) - ***이중 미분 값이 올라갔다 내려갈때를 슬립이 일어난 지점으로 가정하고 슬립 일어난 지점 전 후의 구간을 측정하여 IMU센서 값을 기준으로 로봇의 각도를 보정해줘야 한다.(완료) ->현재 이차 미분값의 변화량을 확인하여 수정해야할 값을 publish 함***
<br>(Issue2) - **drift** 현상 보정
<br>(Issue3) - **IMU**의 틀어진 각도를 다시 보완하는 기능
![image](https://user-images.githubusercontent.com/58541374/161917032-5e5ab851-108d-492c-8527-59d35097d363.png)

<br>(Issue4) - 어차피 **orientation 차이**를 yaw로 바꾸는 것이라 로봇이랑 imu가 동시에 한바퀴를 회전해도 상관은 없을테지만 초기 각도 차이가 클 경우에는 3.14에서 -3.14로 확 바뀔 염려가 있음
예) dθ=3.1 여기에 0.04만 차이가 더해질 시 -3.14로 변한다. 고로 드리프트가 발생했다고 오판할 가능성이 있다. 전 각도차이 -3.12 현 각도차이 3.12 => 실제론 dθ=0.04의 각도가 변했지만 각도차의 변화량 입장에선 dθ=6.2가 변했다고 인식함
<br>(Issue5) - 현재 pose 는 아두이노에서 -> upboard로 일방적으로 받는다고 들음 pose를 수정하기 위해서는 아두이노와 IMU를 연결을 하거나 upboard와 아두이노와 쌍방향 통신이 이루어져야 한다.(코드수정 필요)
<br>(Issue6) - IMU 고정문제가 존재 만약 uart통신이 가능하다면 uart를 시리얼로 바꿔주는 모듈도 필요가 없고 나중에 IMU를 통합보드에 붙여서 사용할 수 있게 됨 (Sensor_IMU_uart 참조) (진행중) 

#시도 BUT 실패

(Issue4 의 대응) -> 아예 현재 dθ 와 과거(1/rate 초 전) dθ 의 Quaternion 차를 구해서 미분값을 구한 후(Quaternion) 이걸 yaw로 바꿔주고
<br>동일하게 이중미분도 현재 dθ' 와 과거(1/rate 초 전) dθ' 의 Quaternion 차를 구해 이중미분 값을 구한 후(Quaternion)으로 바꿔 이걸 yaw로 바꿔줄 생각이였으나
<br>(이 경우 대회전을 하지 않는 이상 미분값이 극과 극으로 뛰는 현상은 방지할 수 있다. -3.14 -> 3.14 => 6.28*30 = 188.4)
<br>너무 작은 Quaternion값을 예) x: 0.0010 y:0 z:0 w:0 yaw로 바꿔줄 경우 -nan값이 관측됨 사용 불가
