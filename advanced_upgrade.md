
## moodle kopija su lxc

Kopija sukuriama su 

```
lxc copy moodle moodle402
```

Paleidžiam

```
lxc start  moodle402

```

IP adresą (jį reikės įrašyti į proxy.moodle.conf vietoj buvusio) pažiūrėti su

```
lxc list  

```

Gauti root teises 

```
sudo - s 

```

Nueiti į

```
cd /etc/nginx/sites-enabled

```
mc

ir surandu kur yra `proxy.moodle.conf`.

Tada

```

cd /etc/nginx/includes/proxy.moodle.conf

```

Šitame faile pakeičiame IP į tą kurį turi naujai sukurtas moodle402 virtualus serveris.

Perkraunam nginx

```

service nginx reload 

```

Reikia prisjungti prie serverio moodle402


```
lxc exec moodle402 bash

```

```
Atsidursiu root

Prisijungiu su user moodle

```
su moodle

```

Eiti į home kataloga

```
cd ~

```

Toliau einu i upgrade_3.9_to_4.0.2 instrukcijas.


Po jų 


Sutvarkome leidimus failams:

 chmod -R ug+rwX /home/moodle/moodle/

 Ir diegiame reikiamus įskiepius.







