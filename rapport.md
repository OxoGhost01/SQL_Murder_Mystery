# BON
On sait juste qu'on a un meurtre à SQL City, aux alentours du 15/01/2018, et on doit trouver le coupable

Commencons donc par chercher dans les reports si il y en a un qui parle d'un meutre

```sql
select * from crime_scence_report
where type = 'murder'
and City = 'SQL City';
```

Cela nous donne :


date | type | description | City
-- | -- | -- | --
20180215|murder|REDACTED REDACTED REDACTED|SQL City
20180215|murder|Someone killed the guard! He took an arrow to the knee!|SQL City
20180115|murder|Security footage shows that there were 2 witnesses. The first witness lives at the last house on "Northwestern Dr". The second witness, named Annabel, lives somewhere on "Franklin Ave".|SQL City

### Nous avons 2 témoins !!

Récupérons leurs `id` et leurs `address_number`

```sql
select * from person
where address_street_name = 'Northwestern Dr'
and address_number = (select max(address_number) from person where address_street_name = 'Northwestern Dr');
```
```sql
select * from person
where address_street_name = 'Franklin Ave'
and name like '%Annabel%';
```

Cela nous donne :


id | name | license_id | address_number | address_street_name | phone_number
-- | -- | -- | -- | -- | --
14887|Morty Schapiro|118009|4919|Northwestern Dr|111564949
16371|Annabel Miller|490173|103|Franklin Ave|318771143

---

### Nous avons donc : 

- Morty, id = 14887
- Annabel, id = 16371

Maintenant qu'on a les contacts + noms des témoins (->leur `id`), allons les interroger

```sql
select * from interview
where person_id = 14887;
```
```sql
select * from interview
where person_id = 16371;
```

Morty nous dit : 
> I heard a gunshot and then saw a man run out. He had a "Get Fit Now Gym" bag. The membership number on the bag started with "48Z". Only gold members have those bags. The man got into a car with a plate that included "H42W".

Et Annabel nous dit :
> I saw the murder happen, and I recognized the killer from my gym when I was working out last week on January the 9th.


Ce que nous dit Morty ici est intéressant, car il nous dit qu'il a eu un bag de "Get Fit Now Gym" et que son numérique de membre commencait par "48Z". On va donc essayer de trouver quel membre a ce numérique de membre.

```sql
select * from get_fit_now_member
where id is like '48Z%'
and membership_status = 'gold';
```

Cela nous donne :   


id |person_id|name|join_date|membership_status
-- | -- | -- | -- | --
48Z7A|28819|Joe Germuska|20160305|gold
48Z55|67318|Jeremy Bowers|20160101|gold

On va essayer de croiser avec la table `person` et le `driver_license` pour confirmer lequel a été vu par Morty

Pour la 1ere peronne (Joe Germuska) : 
- On récupère son `lisense_id` :

```sql
select license_id from person
where id = 28819;
```
-> `173289`
- On regarde sa plaque grâce au `license_id` :

```sql
select plate_number from drivers_license
where id = 173289;
```
-> Rien n'est retouné :/


On passe donc à la 2eme personne (Jeremy Bowers):
- On récupère son `lisense_id` :

    ```sql
    select license_id from person
    where id = 67318;
    ```
    -> 423327
- On regarde sa plaque grâce au `license_id` :

    ```sql
    select plate_number from drivers_license
    where id = 423327;
    ```
    -> OH42W2

## On a donc trouvé la personne dont parlait Morty !

---

## Vérifions que c'est la personne qu'Annabel a dit avoir vu

```sql
select * from get_fit_now_check_in
where membership_id = '48Z55'
and check_in_date = 20180109;
```

On obtient :


membership_id |check_in_date|check_out_date
-- | -- | --
48Z55|20180109|1530|1700


## On a donc trouvé la personne dont parlait Annabel !

# Nous avons trouvé la coupable !!

```sql
select * from person
where id = 67318;
```

Le coupable est : 


id | name | license_id | address_number | address_street_name | phone_number
-- | -- | -- | -- | -- | --
67318|Jeremy Bowers|423327|530|Washington Pl, Apt 3A|871539279

Allons l'arrêter !!

```sql
insert into solution values (1, 'Jeremy Bowers');
```

---

Par curiosité, allons le torturer pour lui demander pour quelle raison il a fait ça :

```sql
select * from interview
where person_id = 67318;
```

> I was hired by a woman with a lot of money. I don't know her name but I know she's around 5'5" (65") or 5'7" (67"). She has red hair and she drives a Tesla Model S. I know that she attended the SQL Symphony Concert 3 times in December 2017.


### Intéréssant, il y avait donc un cerveau et notre Jeremy n'était que l'executeur !

Allons trouver cette femme à l'origine de tout cela.
Pour cela, nous savons qu'elle a assisté à 3 concerts de SQL Symphony, commencons par ici,
puis regroupons cela avec le modèle de sa voiture,
et encore avec sa taille.

```sql
select p.* from person p
join drivers_license dl on p.license_id = dl.id
join facebook_event_checkin fb on p.id = fb.person_id
where dl.height between 65 and 67
and dl.hair_color = 'red'
and dl.car_make = 'Tesla'
and dl.car_model = 'Model S'
and fb.event_name = 'SQL Symphony Concert'
and fb.date between 20171201 and 20171231
group by p.id
having count(fb.event_id) = 3;
```

Et..... on obtient la coupable !!!!


id | name | license_id | address_number | address_street_name | phone_number
-- | -- | -- | -- | -- | --
99716|Miranda Priestly|202298|1883|Golden Ave|987756388

# La coupable est donc Miranda Priestly !

```sql
insert into solution values (2, 'Miranda Priestly');
```