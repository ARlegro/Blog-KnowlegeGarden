---
{"dg-publish":true,"permalink":"/Retrospect/0611~0706/0625 - Design DB and Schema + security❓/","noteIcon":"","created":"2025-12-03T14:52:53.118+09:00","updated":"2025-12-13T18:25:31.866+09:00"}
---

> 지난 일기 [[Retrospect/0611~0706/0624 Help project\|0624 Help project]]


지난 번에 도와주기로 하며 어제 밤에 DB설계를 완성해놨다.
![Clothes-ERD 1.png](/img/user/supporter/image/Clothes-ERD%201.png)
추가로 ERD를 기반으로 스키마도 직접 짜봤다.(Export 기능 없이)
이전에 DB 인덱싱, 동시성 공부할 때 자주 Schema를 수동으로 작성했었는데 기억이 가물가물해서 복습겸 다시 해봤다. PostgreSQL만의 문법(Enum, Text 등)과 Composite PK를 사용해보면서 역시 PostgreSQL.... 좋은 경험을 한 것 같다.
```SQL
CREATE EXTENSION IF NOT EXISTS "pgcrypto";  
  
CREATE TYPE user_role_type AS ENUM ('USER', 'ADMIN');  
CREATE TYPE temperature_sensitivity_type AS ENUM ('ZERO', 'ONE', 'TWO', 'THREE', 'FOUR', 'FIVE');  
CREATE TYPE gender_type AS ENUM ('MALE', 'FEMALE', 'OTHER');  
CREATE TYPE level_type AS ENUM ('INFO', 'WARNING', 'ERROR');  
CREATE TYPE sky_status_type AS ENUM ('CLEAR', 'MOSTLY_CLOUDY', 'CLOUDY');  
CREATE TYPE asWord_type AS ENUM ('WEAK', 'MODERATE', 'STRONG');  
CREATE TYPE precipitation_type AS ENUM ('NONE', 'RAIN', 'RAIN_SNOW', 'SNOW', 'SHOWER');  
CREATE TYPE clothes_type AS ENUM ('TOP', 'BOTTOM', 'DRESS', 'OUTER', 'UNDERWEAR', 'ACCESSORY', 'SHOES', 'SOCKS', 'HAT', 'BAG', 'SCARF', 'ETC');  
  
CREATE TYPE location_type AS (  
    latitude DOUBLE PRECISION,  
    longitude DOUBLE PRECISION,  
    x SMALLINT,  
    y SMALLINT,  
    location_names TEXT[]  
    );  
  
  
CREATE TYPE temperature_type AS (  
    current_value DOUBLE PRECISION,  
    compared_to_day_before DOUBLE PRECISION,  
    min_value DOUBLE PRECISION,  
    max_value DOUBLE PRECISION  
    );  
  
CREATE TYPE precipitation_info_type AS (  
    type precipitation_type,  
    amount DOUBLE PRECISION,  
    probability DOUBLE PRECISION  
    );  
  
CREATE TYPE wind_speed_type AS (  
    speed DOUBLE PRECISION,  
    asWord asWord_type  
    );  
  
CREATE TYPE humidity_type AS (  
    current_value DOUBLE PRECISION,  
    compared_to_day_before DOUBLE PRECISION  
    );  
  
CREATE TABLE users  
(  
    user_id    UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    created_at TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    updated_at TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    password   VARCHAR(255)     NOT NULL,  
    email      VARCHAR(255)     NOT NULL,  
    locked     BOOLEAN          NOT NULL DEFAULT FALSE,  
    role       user_role_type   NOT NULL,  
    is_temporary_password BOOLEAN NOT NULL  
);  
  
CREATE TABLE profiles  
(  
    profile_id              UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    user_id                 UUID             NOT NULL,  
    created_at              TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    updated_at              TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    name                    VARCHAR(255)     NOT NULL,  
    gender                  gender_type,  
    birth_date              DATE,  
    profile_image_url       VARCHAR(1024),  
    location                location_type,  
    temperature_sensitivity temperature_sensitivity_type  
);  
  
  
CREATE TABLE notifications  
(  
    notification_id UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    user_id         UUID             NOT NULL,  
    created_at      TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    title           VARCHAR(50)      NOT NULL,  
    content         VARCHAR(255)     NOT NULL,  
    level           level_type       NOT NULL  
);  
  
  
CREATE TABLE follows  
(  
    follower_id  UUID NOT NULL,  
    following_id UUID NOT NULL,  
    PRIMARY KEY (follower_id, following_id)  
);  
  
  
CREATE TABLE weathers  
(  
    weather_id    UUID PRIMARY KEY   NOT NULL DEFAULT gen_random_uuid(),  
    forecasted_at TIMESTAMP          NOT NULL,  
    forecast_at   TIMESTAMP          NOT NULL,  
    sky_status    sky_status_type    NOT NULL,  
    location      location_type      NOT NULL,  
    precipitation precipitation_type NOT NULL,  
    temperature   temperature_type   NOT NULL,  
    wind_speed    wind_speed_type    NOT NULL,  
    humidity      humidity_type      NOT NULL  
);  
  
CREATE TABLE feeds  
(  
    feed_id    UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    user_id    UUID             NOT NULL,  
    content    TEXT,  
    created_at TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    updated_at TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP  
);  
  
CREATE TABLE feed_comment  
(  
    feed_comment_id UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    feed_id         UUID             NOT NULL,  
    user_id         UUID             NOT NULL,  
    comment         TEXT             NOT NULL,  
    created_at      TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    updated_at      TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP  
);  
  
CREATE TABLE feed_like  
(  
    user_id UUID NOT NULL,  
    feed_id UUID NOT NULL,  
    PRIMARY KEY (user_id, feed_id)  
);  
  
CREATE TABLE clothes  
(  
    clothes_id UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    user_id    UUID             NOT NULL,  
    name       VARCHAR(255)     NOT NULL,  
    image_url  VARCHAR(1024),  
    type       clothes_type     NOT NULL,  
    created_at TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    updated_at TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP  
);  
  
CREATE TABLE feed_clothes  
(  
    feed_id         UUID             NOT NULL,  
    clothes_id      UUID             NOT NULL,  
    PRIMARY KEY (feed_id, clothes_id)  
);  
  
CREATE TABLE attributes  
(  
    definition_id     UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    definition_name   VARCHAR(50)      NOT NULL,  
    selectable_values TEXT[] NOT NULL  
);  
  
CREATE TABLE clothes_attributes  
(  
    clothes_attributes_id UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),  
    clothes_id            UUID             NOT NULL,  
    definition_id         UUID             NOT NULL,  
    value                 VARCHAR(50)      NOT NULL  
);  
  
  
ALTER TABLE users  
    ADD CONSTRAINT uq_users_email UNIQUE (email);  
  
ALTER TABLE profiles  
    ADD CONSTRAINT fk_profiles_users  
        FOREIGN KEY (user_id) REFERENCES users (user_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE notifications  
    ADD CONSTRAINT fk_notifications_users  
        FOREIGN KEY (user_id) REFERENCES users (user_id);  
  
ALTER TABLE follows  
    ADD CONSTRAINT fk_follows_follower_id_users  
        FOREIGN KEY (follower_id) REFERENCES users (user_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE follows  
    ADD CONSTRAINT fk_follows_following_id_users  
        FOREIGN KEY (following_id) REFERENCES users (user_id)  
            ON DELETE CASCADE;  
  
  
ALTER TABLE feeds  
    ADD CONSTRAINT fk_feeds_users FOREIGN KEY (user_id) REFERENCES users (user_id);  
-- ON DELETE CASCADE 넣을지 고민 (user 삭제 시 feed는 남겨 놓을지)  
  
ALTER TABLE feed_comment  
    ADD CONSTRAINT fk_feed_comment_feed  
        FOREIGN KEY (feed_id) REFERENCES feeds (feed_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE feed_comment  
    ADD CONSTRAINT fk_feed_comment_users  
        FOREIGN KEY (user_id) REFERENCES users (user_id)  
            ON DELETE CASCADE;  
  
  
ALTER TABLE feed_like  
    ADD CONSTRAINT fk_feed_like_feed  
        FOREIGN KEY (feed_id) REFERENCES feeds (feed_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE feed_like  
    ADD CONSTRAINT fk_feed_like_users  
        FOREIGN KEY (user_id) REFERENCES users (user_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE clothes  
    ADD CONSTRAINT fk_clothes_users  
        FOREIGN KEY (user_id) REFERENCES users (user_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE feed_clothes  
    ADD CONSTRAINT fk_feed_clothes_feed  
        FOREIGN KEY (feed_id) REFERENCES feeds (feed_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE feed_clothes  
    ADD CONSTRAINT fk_feed_clothes_clothes  
        FOREIGN KEY (clothes_id) REFERENCES clothes (clothes_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE attributes  
    ADD CONSTRAINT uq_attributes_definition_name UNIQUE (definition_name);  
-- selectable_values 값 중복 확인은 그냥 application 계층에서 하기  
  
ALTER TABLE clothes_attributes  
    ADD CONSTRAINT fk_clothes_attributes_clothes  
        FOREIGN KEY (clothes_id) REFERENCES clothes (clothes_id)  
            ON DELETE CASCADE;  
  
ALTER TABLE clothes_attributes  
    ADD CONSTRAINT fk_clothes_attributes_attributes  
        FOREIGN KEY (definition_id) REFERENCES attributes (definition_id)  
            ON DELETE CASCADE;
```

--- 
### 0.1.  솔직한 시간을 갖다
이렇게 DB설계랑, Schema를 짜줬으니 알아서 하겠지?라고 생각했다.
입원한 1명을 제외하고 다른 3명에게 '제가 이제 더 이상 개입하면 안될 것 같으니 이정도만 하고 이제 여러분들이 다들 그라운드 룰, 깃허브 전략 이런 기본적인 룰들부터 정하시면 된다'라고 했다.
근데 문제가 다들 한 3분? 얘기하시더니 전혀 진행이 안되고 있었다???
뭐지??? 일단 팀원들끼리 생각하는게 있으려니 했고, 나는 다시 개인공부를 시작했다.
오후 5시 쯤 걱정이 되어 팀원들에게 진행사항을 여쭤보니.... 하나도 진행된게 없다고 했다.
쩝... 나는 진짜 며칠 뒤면 떠나니 상관없는데 이 사람들 뭐지?? 내가 왜 도와준거지? 이 생각이 들었다.

그래서 이 부분에 대해 얘기하고 다들 솔직하게 프로젝트 하실 맘이 있으신지에 대해 물어보며 앞장 서서 회초리를 들었다. 이렇게 진지하게 말하니 한명이 나와서 파이썬?회사 취업 거의 확정이라고 하시면서 솔직하게 털어 놓으셨다. 역시.. 뭔가 이상했구나 싶었는데 그런거였군 하면서 아 그럼 어쩔 수 없죠..하면서 나머지 분들과도 애기를 꺼냈고. 인원이 부족하다보니 어쩌다보니 내가 프로젝트 초반부인 security를 조금 도와주기로 결정을 했다.
그래.. 뭐 security 복습하고 심화과정 한다고 치고 다시 하면되니까 Let's go...
한명을 제외한 상황 속 다시 역할을 분배해주면서 내일부터 진짜 프로젝트가 진행되기를.... 기도할란다.
