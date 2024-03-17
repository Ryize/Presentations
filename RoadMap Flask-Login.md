# Добавление Flask-Login в свой проект
> \* Этот мануал подходит только для проектов с Flask-SQLAlchemy. Для других ORM он не подойдёт!
<br>

1) Установка Flask-Login
```
pip install flask-login
```

<br><br>2) Настройка app.py (файла, где создаётся переменная app/application)<br>
<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.1) Сначала необходимо импортировать LoginManager и uuid:
```
import uuid
from flask_login import LoginManager
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.2) Далее необходимо добавить SECRET_KEY (после создания переменной app/application) и создать объект manager, далее он нам понадобится: 
```
app.config['SECRET_KEY'] = str(uuid.uuid4())
manager = LoginManager(app)
```

<br><br>3) Изменение models.py (файла, где создаются модели базы данных)<br>
<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.1) Импортируем UserMixin и переменную manager, созданную на предыдущем этапе:
```
from flask_login import UserMixin
from app import manager
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.2) Отнаследуем модель User (модель которая будет представлять пользователя) от UserMixin:
```
class User(db.Model, UserMixin):
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3) Создадим функцию load_user, которая будет отвечать за получение пользователя по id:
```
@manager.user_loader
def load_user(user_id):
    return User.query.get(user_id)
```
<br><br>4) Изменение в controller.py (файл, в котором находятся app.route))<br>
<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.1) Импортируем необходимые функции:
```
from flask import request, redirect, url_for
from flask_login import login_required, login_user, logout_user
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.2) Добавим в конец файла две функции. В них надо изменить текст в url_for. Меняем на **название** функции, куда вы хотите перенаправить пользователя
```
@app.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('index'))


@app.after_request
def redirect_to_sign(response):
    if response.status_code == 401:
        return redirect(url_for('register'))
    return response
```
  <br>
   <hr>

Готово! Теперь вы можете авторизовывать пользователя. Для этого получите экземпляр класса User, и передайте его в login_user. Пример:
```
login = request.form.get('username')
password = request.form.get('password')
user = User(login=login, password=password)
db.session.add(user)
db.session.commit()
login_user(user)
```
