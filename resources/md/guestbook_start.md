[Guestbook 튜토리얼](http://www.luminusweb.net/docs/guestbook.md)은 내용 설명과 따라하는 부분이 섞여 있어 무작정 따라하기에는 약간 어렵다. [Guestbook 튜토리얼](http://www.luminusweb.net/docs/guestbook.md)에서 무작정 따라할 부분만 따로 뽑아 수정함.

## Guestbook Application

### Installing Leiningen

Leiningen 없는 분은 [Leiningen](http://leiningen.org/) 의 내용대로 설치. 

### Creating a new application

```
lein new luminus guestbook +h2
cd guestbook
```

### Creating the Database

`migrations/<date>-add-users-table.up.sql` 수정.

수정전:

```sql
CREATE TABLE users
(id VARCHAR(20) PRIMARY KEY,
 first_name VARCHAR(30),
 last_name VARCHAR(30),
 email VARCHAR(30),
 admin BOOLEAN,
 last_login TIME,
 is_active BOOLEAN,
 pass VARCHAR(100));
```

수정후: 

```sql
CREATE TABLE guestbook
(id INTEGER PRIMARY KEY AUTO_INCREMENT,
 name VARCHAR(30),
 message VARCHAR(200),
 timestamp TIMESTAMP);
```

프로젝트 root 에서 다음 실행.

```
lein ragtime migrate
```

### Accessing The Database

`resources/sql/queries.sql` 수정.

수정전:

```sql
--name: create-user!
-- creates a new user record
INSERT INTO users
(id, first_name, last_name, email, pass)
VALUES (:id, :first_name, :last_name, :email, :pass)

--name: update-user!
-- update an existing user record
UPDATE users
SET first_name = :first_name, last_name = :last_name, email = :email
WHERE id = :id

-- name: get-user
-- retrieve a used given the id.
SELECT * FROM users
WHERE id = :id
```

수정후:

```sql
--name:save-message!
-- creates a new message
INSERT INTO guestbook
(name, message, timestamp)
VALUES (:name, :message, :timestamp)

--name:get-messages
-- selects all available messages
SELECT * from guestbook
```

### Running the Application

실행:

```
>lein run
2015-May-03 10:35:45 -0400 Local INFO [guestbook.handler] -
-=[ guestbook started successfully using the development profile ]=-
2015-05-03 10:35:46.037:INFO:oejs.Server:jetty-7.6.13.v20130916
2015-05-03 10:35:46.069:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:3000
```

웹 브라우저로 실행 결과 확인:
[http://localhost:3000](http://localhost:3000)

이미 3000 포트 사용하고 있는 경우 포트를 바꿔서 실행하려면:

```
lein run 8000
```

### Creating Pages and Handling Form Input

`src/guestbook/routes/home.clj` 를 다음처럼 수정

```clojure
(ns guestbook.routes.home
  (:require [guestbook.layout :as layout]
            [compojure.core :refer [defroutes GET POST]]
            [ring.util.http-response :refer [ok]]
            [guestbook.db.core :as db]
            [bouncer.core :as b]
            [bouncer.validators :as v]
            [ring.util.response :refer [redirect]]))

(defn validate-message [params]
  (first
    (b/validate
      params
      :name v/required
      :message [v/required [v/min-count 10]])))

(defn save-message! [{:keys [params]}]
  (if-let [errors (validate-message params)]
    (-> (redirect "/")
        (assoc :flash (assoc params :errors errors)))
    (do
      (db/save-message! (assoc params :timestamp (java.util.Date.)))
      (redirect "/"))))

(defn home-page [{:keys [flash]}]
  (layout/render
    "home.html"
    (merge {:messages (db/get-messages)}
           (select-keys flash [:name :message :errors]))))

(defn about-page []
  (layout/render "about.html"))

(defroutes home-routes
           (GET "/" request (home-page request))
           (POST "/" request (save-message! request))
           (GET "/about" [] (about-page)))
```

`resources/templates/home.html` 를 다음처럼 수정

```xml
{% extends "base.html" %}
{% block content %}
<div class="row">
    <div class="span12">
        <ul class="messages">
            {% for item in messages %}
            <li>
                <time>{{item.timestamp|date:"yyyy-MM-dd HH:mm"}}</time>
                <p>{{item.message}}</p>
                <p> - {{item.name}}</p>
            </li>
            {% endfor %}
        </ul>
    </div>
</div>
<div class="row">
    <div class="span12">
        <form method="POST" action="/">
                {% csrf-field %}
                <p>
                    Name:
                    <input class="form-control"
                           type="text"
                           name="name"
                           value="{{name}}" />
                </p>
                {% if errors.name %}
                <div class="alert alert-danger">{{errors.name|join}}</div>
                {% endif %}
                <p>
                    Message:
                <textarea class="form-control"
                          rows="4"
                          cols="50"
                          name="message">{{message}}</textarea>
                </p>
                {% if errors.message %}
                <div class="alert alert-danger">{{errors.message|join}}</div>
                {% endif %}
                <input type="submit" class="btn btn-primary" value="comment" />
        </form>
    </div>
</div>
{% endblock %}
```

`resources/css/screen.css` 를 다음처럼 수정

```
body {
	height: 100%;
	padding-top: 70px;
	font: 14px 'Helvetica Neue', Helvetica, Arial, sans-serif;
	line-height: 1.4em;
	background: #eaeaea;
	width: 550px;
	margin: 0 auto;
}

.messages {
  background: white;
  width: 520px;
}
ul {
	list-style: none;
}

li {
	position: relative;
	font-size: 16px;
	padding: 5px;
	border-bottom: 1px dotted #ccc;
}

li:last-child {
	border-bottom: none;
}

li time {
	font-size: 12px;
	padding-bottom: 20px;
}

form, .error {
	width: 520px;
	padding: 30px;
	margin-bottom: 50px;
	position: relative;
  background: white;
}

```

브라우저 페이지 재로딩해서 결과 확인

### 네비게이션 메뉴 디자인 깨진 문제 수정

튜토리얼대로 하면 네비게이션 메뉴 디자인이 약간 깨진다. 

`resources/css/screen.css` 의 일부를 다시 수정한다.

수정전:

```
li {
  position: relative;
  font-size: 16px;
  padding: 5px;
  border-bottom: 1px dotted #ccc;
}
```

수정후:

```
.messages li{
  position: relative;
  font-size: 16px;
  padding: 5px;
  border-bottom: 1px dotted #ccc;
}
```

브라우저 페이지 재로딩해서 결과 확인