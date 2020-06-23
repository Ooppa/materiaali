## Istunnot ja kirjautuminen

Useimmissa web-sovelluksissa on mahdollista kirjautua sisään antamalla tunnus ja salasana, minkä jälkeen pääsee tekemään jotain erityistä. Kirjautuminen vaatii, että tietoa säilyy sovelluksessa sivulta toiselle, ja tämä onnistuu _istunnon_ (_session_) avulla.

Ideana on, että voimme tallentaa istuntoon avain-arvo-pareja, jotka säilyvät muistissa sivulta toiselle. Flask toteuttaa istunnon `session`-oliona, jonka tiedot tallennetaan selaimeelle lähetettävään _evästeeseen_ (_cookie_).

Istunnon käyttäminen vaatii, että sovelluksessa on käytössä salainen avain. Lisäämme satunnaisesti muodostetun salaisen avaimen `.env`-tiedostoon:

```bash
SECRET_KEY=WMyHhYFPTpQ9geoDboUXDUZ9f6wMOrjO
```

Tämä avain allekirjoittaa evästeessä olevan tiedon niin, että käyttäjä ei pysty muuttamaan istunnon sisältöä selaimessa. **On tärkeää, että avain on salainen, eli älä missään tapauksessa käytä yllä olevaa avainta vaan luo oma salainen avain!**

Jos haluat tietää tarkemmin, miten istunto on toteutettu, voit lukea lisää [tästä](TODO).

### Kirjautumisen toteutus

Seuraava sovellus antaa näytteen kirjautumisen toteuttamisesta:

```python
from flask import Flask, render_template, request, redirect, session
from dotenv import load_dotenv
import os

app = Flask(__name__)
app.secret_key = os.getenv("SECRET_KEY")

@app.route("/")
def index():
    username = session.get("username","")
    return render_template("index.html",username=username)

@app.route("/login",methods=["post"])
def login():
    username = request.form["username"]
    password = request.form["password"]
    # TODO: check username and password
    session["username"] = username
    return redirect("/")

@app.route("/logout")
def logout():
    del session["username"]
    return redirect("/")
```

Sovellus käyttää seuraavaa sivupohjaa `index.html`, joka joko näyttää kirjautumislomakkeen tai kertoo, että käyttäjä on sisällä:

```html
{% raw %}{% if username == "" %}
<form action="/login" method="post">
<p>Tunnus:<br>
<input type="text" name="username"></p>
<p>Salasana:<br>
<input type="password" name="password"></p>
<input type="submit" value="Kirjaudu">
</form>
{% else %}
<p>Olet kirjautunut nimellä {{ username }}</p>
<a href="/logout">Kirjaudu ulos</a>
{% endif %}{% endraw %}
```

Sovelluksen käyttäminen voi näyttää tältä:

TODO: Kuva tähän

Katsotaan nyt tarkemmin sovelluksen osia: 

```python
app.secret_key = os.getenv("SECRET_KEY")
```

Tällä rivillä sovellus hakee salaisen avaimen ympäristömuuttujasta `SECRET_KEY`. Istuntoa ei ole mahdollista käyttää ilman salaista avainta.

```python
@app.route("/")
def index():
    username = session.get("username","")
    return render_template("index.html",username=username)
```

Etusivulla sovellus hakee `session`-rakenteesta käyttäjän tunnuksen avaimesta `username`. Jos avainta ei ole (eli käyttäjä ei ole kirjautuneena), `get`-metodi palauttaa oletusarvona tyhjän merkkijonon. Sivupohja päättelee muuttujasta `username`, onko käyttäjä kirjautunut vai ei.

```python
@app.route("/login",methods=["post"])
def login():
    username = request.form["username"]
    password = request.form["password"]
    # TODO: check username and password
    session["username"] = username
    return redirect("/")
```

Tämä funktio käsittelee kirjautumislomakkeen. Tällä hetkellä funktiosta puuttuu vielä osa (TODO-kohta), joka varmistaa, että käyttäjä antaa oikean tunnuksen ja salasanan. Niinpä sovellus hyväksyy minkä tahansa tunnuksen ja salasanan.

```python
@app.route("/logout")
def logout():
    del session["username"]
    return redirect("/")
```

Tämä funktio kirjaa käyttäjän ulos poistamalla `session`-rakenteesta avaimen `username`.

### Käyttäjät tietokannassa

Kirjautuminen on järkevää toteuttaa niin, että tiedot käyttäjistä ovat tietokannassa. Voimme käyttää tähän seuraavan tapaista taulua:

```sql
CREATE TABLE users (id SERIAL PRIMARY KEY, username TEXT, password TEXT);
```

Tässä tapauksessa käyttäjästä tallennetaan käyttäjätunnus ja salasana. Tämän lisäksi taulussa voisi olla monenlaista muutakin tietoa, kuten onko käyttäjä peruskäyttäjä vai admin-käyttäjä.

Turvallinen tapa tallentaa salasana tietokantaan on tallentaa selkokielisen salasanan sijasta salasanan _hajautusarvo_ (_hash value_), jonka avulla voidaan tarkastaa, onko salasana oikea. Voimme käyttää tähän Flaskin mukana tulevaa Werkzeug-kirjastoa:

```python
import werkzeug.security
```

Esimerkiksi seuraava koodi lisää tietokantaan uuden käyttäjän, jonka tunnus on `username` ja salasana on `password`. Koodi laskee salasanan hajautusarvon Werkzeug-kirjaston funktiolla `generate_password_hash`.

```python
hash_value = werkzeug.security.generate_password_hash(password)
sql = "INSERT INTO users (username,password) VALUES (:username,:password)"
db.session.execute(sql, {"username":username,"password":hash_value})
db.session.commit()
```

Seuraava koodi puolestaan tarkastaa, onko käyttäjän antama tunnus ja salasana oikein:

```python
sql = "SELECT password FROM users WHERE username=:username"
result = db.session.execute(sql, {"username":username})
hash_value = result.fetchone()    
if hash_value == None:
    # TODO: invalid username
else:
    if werkzeug.security.check_password_hash(hash_value[0],password):
        # TODO: correct username and password
    else:
        # TODO: invalid password
```

Koodi hakee tietokannasta käyttäjän antamaa tunnusta vastaavan salasanan. Jos tietokannasta ei tule riviä (tulos on `None`), tämä tarkoittaa, että tietokannassa ei ole käyttäjätunnusta. Muuten tarkastetaan Werkzeug-kirjaston funktiolla `check_password_hash`, onko salasana oikea.

Salasana näyttää tietokannassa seuraavalta:

```bash
pllk=# SELECT password FROM users WHERE username='maija';
                                            password                                            
------------------------------------------------------------------------------------------------
 pbkdf2:sha256:150000$98mnxMjT$cedf43db2f098831ab5f533d814cf094db01540e34251aee3d5afd7d5607fc5a
(1 row)
```

Tässä tapauksessa käyttäjän "maija" salasana on "kissa". Salasanasta on tallennettu tietokantaan merkkijono, jonka osana on käytetty hajautusfunktio (tässä `sha256`), muuta tietoa hajautustavasta ja varsinainen hajautusarvo. Tämän avulla voidaan tarkastaa myöhemmin, onko käyttäjän antama salasana oikea.