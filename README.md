## Docker環境用のファイルを作成する
Dockerfile.dev

```
FROM ruby:3.2.3
ENV LANG C.UTF-8
ENV TZ Asia/Tokyo
RUN apt-get update -qq \
&& apt-get install -y ca-certificates curl gnupg \
&& mkdir -p /etc/apt/keyrings \
&& curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
&& NODE_MAJOR=20 \
&& echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list \
&& wget --quiet -O - /tmp/pubkey.gpg https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
&& echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
RUN apt-get update -qq && apt-get install -y build-essential libssl-dev nodejs yarn vim
RUN mkdir /myapp
WORKDIR /myapp
RUN gem install bundler
COPY . /myapp
```

compose.yml

```
services:
  db:
    image: mysql:8.0
    environment:
      TZ: Asia/Tokyo
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - 3307:3306
    healthcheck:
      test: mysqladmin ping -h 127.0.0.1 -uroot -ppassword
      interval: 10s
      timeout: 10s
      retries: 3
      start_period: 30s
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bash -c "bundle install && bundle exec rails db:prepare && rm -f tmp/pids/server.pid && ./bin/dev"
    tty: true
    stdin_open: true
    volumes:
      - .:/myapp
      - bundle_data:/usr/local/bundle:cached
      - node_modules:/myapp/node_modules
    environment:
      TZ: Asia/Tokyo
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
volumes:
  mysql_data:
  bundle_data:
  node_modules:
```

## rails new

### Bootstrapを使用する場合

```
docker compose build
docker compose run --rm web gem install rails
docker compose run --rm web rails new . -d mysql -j esbuild --css=bootstrap
```

### Tailwind CSSを使用する場合

```
docker compose build
docker compose run --rm web gem install rails
docker compose run --rm web rails new . -d mysql -j esbuild --css=tailwind
```

### サーバーを立ち上げる前の準備

config/database.ymlのdefaultでhostをdbにpasswordにpassword(compose.ymlの内容と合わせます)を設定

```
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password: password
  host: db
```

Procfile.devのwebに`-b 0.0.0.0 -p 3000`をつける

```
web: env RUBY_DEBUG_OPEN=true bin/rails server -b 0.0.0.0 -p 3000
js: yarn build --watch
css: yarn build:css --watch
```

### サーバーを立ち上げる

```
docker compose up
```

### ブラウザで確認

http://localhost:3000
