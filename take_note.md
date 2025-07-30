## 🌹 Triển khai CAS (Central Authentication Service) bằng Docker trên EC2 Linux

### 🍊 0. Một số khái niệm có liên quan đến demo cần biết
- CAS là một hệ thống xác thực tập trung. Nó cho phép nhiều ứng dụng (client) cùng sử dụng một hệ thống đăng nhập duy nhất. Tức là:
  - Bạn chỉ cần đăng nhập CAS một lần, khi bạn login vào _https://app1.example.com_ thành công, sau đó vào _https://app2.example.com_ mà không cần login lần nữa.
  - Khi nào CAS yêu cầu đăng nhập lại?
    - Khi bạn logout khỏi CAS
    - Khi ticket/SSO session hết hạn (ticket: Mã xác thực tạm thời được CAS cấp cho mỗi phiên).
    - Khi dùng ```renew=true``` hoặc ```gateway=true``` trong URL login.
- ```gradlew```:
  - Viết đầy đủ là Gradle Wrapper, là một file script được dùng để build ứng dụng Java sử dụng Gradle, mà không cần cài Gradle sẵn trên máy.
  - File:
    - ./gradlew: Script chạy Gradle cho Linux/macOS
    - gradlew.bat: Script chạy Gradle cho Windows
    - gradle/wrapper/:	Chứa file cấu hình tải đúng phiên bản Gradle
  - Trong CAS Overlay, ```./gradlew build``` được dùng để build ra file _build/libs/cas.war_, file war này sau đó sẽ được dùng để chạy CAS.
- So sánh file: _src/main/resources/application.yml_ vs _./config/application.yml_:
  | Path | Ưu tiên load	| Trường hợp sử dụng | Ghi đè được không?|
  |------|--------------|--------------------|-------------------|
  |src/main/resources/application.yml| Thấp hơn | Dùng cho cấu hình mặc định, khi build thành .war | Có thể bị ghi đè|
  | ./config/application.yml | Cao hơn | Dùng để override cấu hình mặc định, runtime config	| Ghi đè toàn bộ các config khác |

  - Nên dùng _./config/application.yml_ vì: cấu hình linh hoạt, dễ test, dễ override mà không cần rebuild Docker images.

- Phân biệt thuộc tính ```name``` và ```prefix``` của server:
  - ```server.name```: Địa chỉ gốc (root path) của CAS server dùng để sinh URL redirect/ticket => Không phải URL để truy cập trực tiếp.
  - ```server.prefix```: Đường dẫn đầy đủ đến endpoint của CAS => Là endpoint chính để người dùng truy cập.
> Lưu ý:
> Chúng ta sẽ truy cập CAS qua ```http://<IP_public_of_EC2>:8080/cas/``` thay vì ```http://<IP_public_of_EC2>/``` (root path không có gì để hiển thị).
  
### 🍊 1. Chuẩn bị
- Kiểm tra OS để chọn bản phù hợp: ```cat /etc/os-release```
- Cài đặt JDK 21.
- Cài Docker.

### 🍊 2. Clone repo CAS và build WAR
- Clone CAS Overlay Template: ```git clone https://github.com/apereo/cas-overlay-template.git``` -> ```cd cas-overlay-template ```
- Build cục war: ``` ./gradlew clean build -x test```
  - ❓ Khi nào cần build lại file WAR?
    - Bạn sửa code nguồn (Java, config, tài nguyên trong src/...)
    - Hoặc sửa build.gradle, application.yml trong source, các dependency...
    - Muốn cập nhật logic, chức năng mới trong ứng dụng.
- Copy cục war ra ngoài thư mục chứa Dockerfile (hoặc không cp ra thì check cấu hình xem path đúng chưa): ```cp build/libs/cas.war .```
- Cấu hình trong file _./config/application.yml_:
  
  ```yml
  # Application properties that need to be
  # embedded within the web application can be included here
  server:
    address: 0.0.0.0 # CAS sẽ lắng nghe trên mọi địa chỉ IP (cả localhost và public IP) - dùng cho trường hợp truy cập URL của cas từ 
    port: 8080        # CAS chạy ở cổng 8080.
    ssl:
      enabled: false   # Tắt HTTPS, chỉ dùng HTTP
  cas:
    server:
      name: http://<IP_public_of_EC2>:8080
      prefix: http://<IP_public_of_EC2>:8080/cas
    authn:
      accept:
        enabled: true        # cờ "true" cho phép dùng user đơn giản khai báo sẵn như user1::Apple để login.
        users: user1::Apple  # account dùng để test việc login
  
  ```

> Lưu ý:
> Nếu dùng file _src/main/resources/application.yml_, sau khi chỉnh sửa file cấu hình thì phải chạy lại 2 câu lệnh trên để build ra cục war mới.
  
### 🍊 3. Build Docker
- Viết Docker file (tôi dùng Dockerfile được cung cấp sẵn trong repo, chỉ sửa lại các trường sau cho phù hợp):
  ```Dockerfile
  ARG BASE_IMAGE="amazoncorretto:21"
  FROM $BASE_IMAGE AS overlay
  COPY cas.war /cas.war
  EXPOSE 8080 8443
  ENTRYPOINT ["java", "-jar", "/cas.war"]
  ```
- Build Docker image:
  ```shell
  ./gradlew clean build
  docker build -t my-demo-cas .
  ```
  > Lưu ý:
  > Khi chỉnh Dockerfile, để container chạy theo Dockerfile mới, phải build lại image, stop container cũ và run lại container từ image vừa build.
  
- Chạy container Docker:
  ```shell
  sudo docker run --rm -d -p 8080:8080 -v ./config:/etc/cas/config my-demo-cas
  # docker run: Chạy một container mới.
  # --rm: Tự động xoá container sau khi nó dừng. Không để lại container cũ.
  # -d: Detached mode — chạy nền (background).
  # -p 8080:8080: Map port 8080 của máy host (EC2) sang port 8080 của container
      # → Cho phép truy cập CAS từ ngoài qua http://<host>:8080/cas.
  # -v ./config:/etc/cas/config: Mount thư mục ./config trên host EC2 vào /etc/cas/config trong container.
      # → Container sẽ dùng file cấu hình application.yml, services/*.json, v.v. từ ./config.
  # my-demo-cas: Tên Docker image đã build trước.
  ```

  > Lưu ý:
  > Cách mount thư mục này sẽ không phù hợp để phát triển dự án lâu dài vì: dev khác không có file _./config_ thì container lỗi, khó CI/CD.
  > Do đó nên đóng gói config files vào image rồi build luôn (thực hiện phía bên dưới).
  
### 🍊 4. Kiểm tra CAS đã hoạt động hay chưa
- Kiểm tra từ bên trong EC2: ```curl -i http://localhost:8080/cas```
- Kiểm tra từ bên ngoài: _http://<IP_public_of_EC2>/cas/_
  - Để check được IP public của EC2, chạy lệnh: ```curl ifconfig.me ```
- Login bằng user đã khai báo trong _application.yml_

## 🌹 Đóng gói cấu hình CAS vào Docker image
### 🍊 1. Hướng thực hiện
- Tạo thư mục chứa config CAS gồm file .properties, .json,...
- Viết Dockerfile để copy config đó vào đúng vị trí trong image.
- Build image riêng và dùng image này để chạy container.

### 🍊 2. Thực hiện
- Cây thư mục sẽ như thế này:
  ```
  ssl_Docker_cas/
  ├── Dockerfile
  └── config/
      └── application.yml
  ```
- Sửa Dockerfile: copy folder cấu hình _config/_ vào image CAS: thêm lệnh ```COPY config /etc/cas/config```

  ```yml
  #------------ BUILD STAGE ---------------
  # Image Gradle with JDK 21
  FROM gradle:8.7-jdk21 AS build
  
  # Install Git
  RUN apt-get update && apt-get install -y git
  
  WORKDIR /cas-overlay-template
  RUN git clone https://github.com/apereo/cas-overlay-template.git .
  
  RUN ./gradlew clean build -x test
  
  #------------ RUNTIME STAGE -------------
  FROM openjdk:21-jdk-slim AS runtime
  
  WORKDIR /cas-overlay-template
  
  COPY --from=build /cas-overlay-template/build/libs/cas.war ./cas.war
  
  COPY config /etc/cas/config
  
  RUN keytool -genkeypair \
      -alias cas \
      -keyalg RSA \
      -keysize 2048 \
      -storetype PKCS12 \
      -keystore cas.p12 \
      -storepass <pass> \
      -keypass <pass> \
      -validity 365 \
      -dname "CN=localhost, OU=Dev, O=MyOrg, L=City, S=State, C=US"
  
  EXPOSE 8443
  
  ENTRYPOINT ["java", "-jar", "cas.war",\
    "--server.ssl.key-store=cas.p12",\
    "--server.ssl.key-store-password=<pass>",\
    "--server.ssl.key-password=<pass>",\
    "--server.ssl.key-store-type=PKCS12",\
    "--server.port=8443"]
    ```
- Build image: ```docker build -t cas-application .```
- Run container từ image vừa build: ```docker run --rm -d -p 8443:8443 --name cas cas-application```

## 🌹 Hướng tiếp theo: Tích hợp CAS với LDAP

Khi người dùng đăng nhập vào CAS → CAS sẽ xác thực thông tin qua LDAP server.
