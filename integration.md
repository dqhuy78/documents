# Integrate FramgiaCI into your Project

## framgia-ci.yml

Một yêu cầu thiết lập quan trọng để có thể chạy framgia-ci là bạn phải có file cấu hình `framgia-ci.yml` trong project của bạn. Bạn có thể tạo một file bằng cách thủ công với nội dụng như dưới đây. Hoặc có thể [Framgia CI CLI Tool](https://github.com/framgiaci/framgia-ci-cli) và sử dụng tool này để tự động sinh file `framgia-ci.yml`. 

```yml
project_type: php
build:
  general_test:
    image: framgiaciteam/laravel-workspace:latest
    services:
      mysql_test:
        image: mysql:5.7
        environment:
          MYSQL_DATABASE: homestead
          MYSQL_USER: homestead
          MYSQL_PASSWORD: secret
          MYSQL_ROOT_PASSWORD: root
      mogodb:
        image: mongo:latest
        environment:
          MONGO_INITDB_ROOT_USERNAME: root
          MONGO_INITDB_ROOT_PASSWORD: mogo_test
    prepare:
      - cp .env.civ3.example .env
      - php artisan config:clear
      - composer install
      - framgia-ci test-connect mysql_test 3306 60
      - php artisan migrate --database=mysql_test
test:
  eslint:
    ignore: false
    command: eslint --format=checkstyle
      --output-file=.framgia-ci-reports/eslint.xml
      resources/assets/js/ --ext .js
  phpcpd:
    ignore: true
    command: phpcpd --log-pmd=.framgia-ci-reports/phpcpd.xml app
  phpmd:
    ignore: true
    command: phpmd app xml cleancode,codesize,controversial,design,naming,unusedcode --reportfile .framgia-ci-reports/phpmd.xml
  pdepend:
    ignore: true
    command: pdepend --summary-xml=.framgia-ci-reports/pdepend.xml
      --jdepend-chart=.framgia-ci-reports/pdepend.svg
      --overview-pyramid=.framgia-ci-reports/pyramid.svg
      app
  phpmetrics:
    ignore: true
    command: phpmetrics --report-html=.framgia-ci-reports/metrics.html
      --report-xml=.framgia-ci-reports/metrics.xml
      app
  phpcs:
    ignore: false
    command: echo '' | phpcs --standard=Framgia --report-checkstyle=.framgia-ci-reports/phpcs.xml --ignore=app/Supports/* app
  phpunit:
    ignore: false
    command:
      - php -dzend_extension=xdebug.so vendor/bin/phpunit
        --coverage-clover=.framgia-ci-reports/coverage-clover.xml
        --coverage-html=.framgia-ci-reports/coverage
deploy:
  rocketeer:
    image: framgiaciteam/deployer:2.0
    when:
      branch: master
    run: php rocketeer.phar deploy --on=staging --no-interaction
cache:
  composer:
    folder: vendor
    file: composer.lock
  nodejs:
    folder: node_modules
    file: yarn.lock
```

1. `project_type` <br/>
    Kiểu project của bạn. Ví dụ như `php`, `ruby`, `android`..v.v.
2. `build` <br/>
    Đây sẽ là nơi bạn chạy các tools, câu lệnh test cho project của bạn.
    * `general_test` Tên của phần chạy test này. Có thể là `unit_test`, `feature_test`, bất kỳ keyword nào tùy ý.
        + `image: framgiaciteam/laravel-workspace:latest` <br/>
        Tên `image` mà bạn sẽ sử dụng để start lên `containers` lên và chạy các câu lệnh test trong đó. Ở ví dụ trên là image `framgiaciteam/laravel-workspace:latest`. Bạn có thể tạo một docker image. Public nó lên Dockerhub và sử dụng nó cho project của bạn 
         `image: framgiaciteam/my_docker_image:1.1.0`
            > Vui lòng chỉ rõ image tag. Nếu không, framgia-ci mặc định sẽ sử dụng tag `latest` cho image của bạn
        + `services` <br/>
        Project của bạn cần kết nối đến database để thực hiện chạy test (unit test, test migrate, test connect db...) 
        Đây là nơi bạn cần định nghĩa các services cần thiết đó.
        Ờ file `framgia-ci.yml` mẫu trên. Ứng dụng Laravel cần test kết nối đến `mysql` database và chạy thử migrate. Nên chúng ta định nghĩa một service tên là `mysql_test`: <br/>
            ```yml
                services:
                    mysql_test:
                        image: mysql:5.7
                        environment:
                        MYSQL_DATABASE: homestead
                        MYSQL_USER: homestead
                        MYSQL_PASSWORD: secret
                        MYSQL_ROOT_PASSWORD: root
            ```
            Trong đó: <br/>
            * `mysql_test` là tên service
            * `image` - Tên docker image mà chúng ta sẽ start service container lên
            * `environment` - Là các biến môi trương `ENV` mà chúng ta cần cho container. Các environments cần thiết này bạn có thể tìm thấy ở phần readme của các docker image. Ví dụ mysql image chẳng hạn: [https://hub.docker.com/_/mysql/](). Trường hợp bạn tự build image thì chắc bạn sẽ nắm rõ các biến môi trường này rồi.
                services:
            > Quan trọng ! Hãy thực hiện kết nối đến services thông qua tên services ! (Tương tự như khi bạn dùng docker compose vậy). Ví dụ tên service là `mysql_test` thì trong file .env phần config connect database bạn đổi database host name thành tên service `DB_HOST=mysql_test`
        + `prepare` Tập các câu lệnh chạy mà bạn muốn chạy trên project của bạn. Ví dụ bạn muốn chạy composer `install` chẳng hạn: <br/>
            ```
                - composer install
            ```

3. `deploy` Đây là nơi bạn sẽ định nghĩa các plugin của bạn.
    ```yml
    rocketeer:
    image: framgiaciteam/autodeploy:latest
    when:
        branch: master
    run: php rocketeer.phar deploy --on=staging --no-interaction
    ```
    + `rocketeer`: Tên plugin
    + `image`: Plugin image
    +  `when`: Định nghĩa khi nào pluing của bạn sẽ chạy<br/>
        + `branch:master`: Chạy khi build ở branch master
        + `status:success`: Chạy khi bản build `success`
    + `run`: Câu lệnh bạn sẽ chạy bên trong plugin container
4. `Cache`: <br/>
    FramgiaCI chỉ thực hiện cache khi folder bạn cần cache thực sự thay đổi. Ví dụ
    ```yml
        cache:
        git:
            folder: .git
        composer:
            folder: vendor
            file: composer.lock
        nodejs:
            folder: node_modules
            file: yarn.lock
    ```
    Ví dụ trên đây dành cho một project Laravel. Các folder như `vendor`, `node_modules` chỉ thay đổi khi bạn thêm hoặc bỏ bớt một hay nhiều packages ra khỏi project. Những thay đổi này được ghi lại ở các file tương ứng là `composer.lock` (compose install) và `yarn.lock` (yarn install). FramgiaCI quan sát sự thay đổi trên các file này để quyết định cache lại folder bằng phiên bản mới hay không.
    Trường hợp bạn muốn cache folder mà không có file tương ứng. Thì chỉ chỉ đơn giản là điền folder thôi và không cần file nữa:
    ```yml
        git:
            folder: .git
    ``` 
* `test` Tham khảo tại [Framgia CI CLI Tool Documents](https://github.com/framgiaci/framgia-ci-cli)

## Một số lưu ý khi active và setting projects

### Active project và đồng bộ member
- Để active projects bạn vào `Available repository` -> `Fetch repositories` -> Chọn repository cần active -> Nhấn `Create Hook`.
        > Lưu ý, chỉ admin của repository mới có quyền active projects

- Sau khi bạn đã active project. Bạn vào trang `Detail` -> `setting` và nhấn `Re - SYNC REPO'S MEMBER`. Trong trường hợp các thành viên còn lại không xem được thông tin các bản build.

### Nhận thông báo về kết quả bản build
- Mặc định khi bạn active một project, một default chatbot của hệ thống được thêm vào project của bạn. Tuy nhiên, để nhận được thông báo từ default chatbot. Bạn cần bật nhận thông báo chatwork. Bạn vào `Detail` -> `NOTIFICATION` -> Bật `CHATWORK NOTIFICATION` và thêm chatwork chatroom ID mà bạn muốn chatbot gửi thông báo vào đó, trong phần `Room Lists`
- Ngoài ra, bạn có thể tự tạo chatbot vào thêm chatbot key vào phần `Chatwork Bot`
- Nhận thông kết quả chạy build qua email: Để nhận thông báo qua email, vẫn trong phần `SETTING -> NOTIFICATION` này, bạn thêm email của người nhận vào phần `Email Lists`
- Nhận thông báo qua slack: Để nhận thông báo qua slack, bạn bật thông báo slack lên. Và thêm các thông tin cần thiết vào mục `Slack Notification`

### Public key
- FramgiaCI sẽ cung cấp cho bạn một public key. Bạn có thể dùng public key này để thêm vào server của bạn (Trường hợp bạn muốn sử dụng service auto deploy của FramgiaCI chẳng hạn, bạn sẽ cần đến public key này). Bạn vào phần `Detail` -> `SETTING` -> `SECRET`. Copy và sử dụng public key này...

## Một số lưu ý khi settup project Ruby
### Cache Gems
- Project ruby thường install gems vào /usr/local/bin/gems và share thư mục gems này giữa các project trên một máy host. Trong khi đó, FramgiaCi cô lập môi trường giữa các bản build khiến các bản build của cùng một project hoàn toàn tách biệt với các bản build còn lại. Nên không thể giữ lại folder share gem này trong các containers. Vậy nên, khi settup một project Ruby bạn hãy cài gem vào trong project folder bằng cách chỉ định install path như dưới đây
        ```bash
            bundle install --path vendor/bundle
        ```
### Cài đặt rspec, brakeman, rake, reek...
- Một số project gặp trường hợp đã chạy bundle install thành công nhưng rspec báo not foud. Nếu gặp trường hợp này, bạn hãy cài thằng các gems đó vào Dockerfile. Như ví dụ dưới đây:
```
FROM ruby:2.4.0
RUN apt-get install -y libpq5 libpq-dev
RUN curl -sL https://deb.nodesource.com/setup_8.x | bash -
RUN apt-get install -y nodejs
# Install eslint from https://www.npmjs.com/package/eslint
RUN npm install -g eslint
RUN eslint --version
RUN apt-get -y update && apt-get -y install ruby-full
RUN gem install rspec \
    scss_lint \
    brakeman \
    bundle-audit \
    reek \
    rails_best_practices \
    simplecov \
    robocop \
    rake
WORKDIR /
```
