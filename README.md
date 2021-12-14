# Grundkompetenz Datenbanken LE5: Bericht

## Zusammenfassung

### Welche Datenbank hast Du gewählt? 

Wir haben uns für MongoDB entschieden.

### Warum hast Du diese gewählt?

Wir haben uns für MongoDB entschieden weil MongoDB viele gut strukturierte Testdaten bereitstellt. Wir haben bis jetzt noch nie mit Dokumentdatenbanken gearbeitet und wollten somit unsere Kompetenzen erweitern. 

### Wie sehen die Daten aus?

Die Daten werden von MongoDB als json Strings zurückgegeben.
Die Tabellen heissen unter MongoDB "Collections" und unterscheiden sich von herkömmlichen Tabellen in relationalen Datenbanken, indem sie kein Datenstrakturschema vorgeben.

### Inwiefern ist das besser als eine relationale Datenbank?

Da die Daten kein Schema haben, können Änderungen bei der Datenerfassung leichter umgesetzt werden. Für grosse Datenservices lohnt sich MongoDB, da die Datenbank horizontal skalierbar ist (mehrere Nodes) und die Latenz und Auslastung weltweit minimiert werden kann. Bei objektorientiertem Code kann ein Objekt zusätzlich schnell geparst und weiterverarbeitet werden.

### Wie sehen komplexe Fragestellungen (Abfrage) zu den Daten aus, und warum sind sie komplex? Vergleiche es mit eine SQL, würde es komplizierter sein, welche Vorteile gibt es gegenüber SQL?

Bei komplexen Abfragen werden die Daten nicht nur abgelesen, sondern stark verarbeitet (Datenaufbereitung), bereinigt, gefiltert und mit anderen Daten kombiniert.

## Abfragen und Resultate

### Anzahl Kommentare aller Filme zurückgeben, die mindestens 150 Awards gewonnen haben.
    {'anzahlKommentare': 267}
Wenn man alle Kommentare von den Filmen, die mindestens 150 Awards gewonnen haben aufsummiert, dann erhält man 267 Kommentare.

### Welche Benutzer haben bei Filme mit 100 oder mehr Awards die meisten Kommentare zum gleichen Film erfasst?
    {'countComments': 5,
     'movie': {'title': 'American Beauty'},
     'user': {'name': 'Daenerys Targaryen'}}
    {'countComments': 5,
     'movie': {'title': 'American Beauty'},
     'user': {'name': 'Kathryn Sosa'}}
     ...
Daenerys Targaryen hat 5 Kommentare zum Film American Beauty geschrieben. Kathryn Sosa hat ebenfalls 5 Kommentare zum Film American Beauty hinterlassen.

### Welche Stadt hat die meisten "Theaters" und in welchem Staat befindet sich die Stadt?
    {'city': 'Las Vegas', 'countTheaters': 29, 'state': 'NV'}
    {'city': 'Houston', 'countTheaters': 22, 'state': 'TX'}
    ...
Die Stadt mit den meisten Theaters ist Las Vegas. Las Vegas hat 29 Theaters. Las Vegas befindet sich im Staat NV (Nevada).

### In welchem Land wurden die meisten Filme gedreht und wieviel Nominierungen haben diese Filme insgesamt erhalten.
    {'countMovies': 10403, 'countNominations': 96974, 'country': 'USA'}
    {'countMovies': 1980, 'countNominations': 18916, 'country': 'UK'}
    ...
Die meisten Filme wurden in der USA gedreht. In der USA wurden 10403 Filme gedreht. Die 10403 Filme haben insgesamt 96974 Awardnominierungen.

### Welches Genre hat die meisten Nominierungen pro Film? (Filme vom letztem Jahrtausend)
    {'countMovies': 410,
     'countNominations': 5201,
     'genre': 'Biography',
     'nominationsPerFilm': 12.685365853658537}
    {'countMovies': 6,
     'countNominations': 45,
     'genre': 'History',
     'nominationsPerFilm': 7.5}
     ...
Filme vom Genre Biography haben die meisten Nominierungen pro Film. Im Durchschnitt hat ein Film vom Genre Biography 12.7 Nominierungen.

### Die besten Filme nach unserem Film-Score. Voraussetzung: Mindestens 10 User Bewertungen
    {'customScore': 1392.0,
     'plot': 'A medical engineer and an astronaut work together to survive after a '
             'catastrophe destroys their shuttle and leaves them adrift in orbit.',
     'title': 'Gravity',
     'year': 2013}
     ...
Nach unserem Filmscore hat der Film Gravity den grössten Score. Gravity erzielt bei uns 1392 Punkte.

# Grundkompetenz Datenbanken LE5: Anhang

## Setup

Hier werden die benötigten Daten von der Konfigurationsdatei abgelesen, um eine Datenbankverbindung herzustellen.


```python
from pymongo import MongoClient
import configparser
import pprint

jsonp = pprint.pprint
```


```python
config = configparser.ConfigParser()
config.read('config.ini')

db_username = config['Database']['USER']
db_password = config['Database']['PASS']
db_hostname = config['Database']['HOST']
```

Hier wird die Datenbankverbindung hergestellt.


```python
client = MongoClient("mongodb+srv://{USERNAME}:{PASSWORD}@{HOSTNAME}".format(USERNAME = db_username, 
                                                                             PASSWORD = db_password, 
                                                                             HOSTNAME = db_hostname))
db = client.sample_mflix
```

Testabfrage:


```python
print(db.list_collection_names())
```

    ['sessions', 'movies', 'users', 'comments', 'theaters']


## Komplexe Abfragen

Anzahl Kommentare aller Filme zurückgeben, die mindestens 150 Awards gewonnen haben.


```python
result = db.movies.aggregate([
    {
        '$addFields': {
            'movie_id': {
                '$toString': '$_id'
            }
        }
    }, {
        '$lookup': {
            'from': 'comments', 
            'localField': '_id', 
            'foreignField': 'movie_id', 
            'as': 'comments'
        }
    }, {
        '$addFields': {
            'countComments': {
                '$size': '$comments'
            }
        }
    }, {
        '$match': {
            'countComments': {
                '$gt': 0
            }, 
            'awards.wins': {
                '$gt': 149
            }
        }
    }, {
        '$group': {
            '_id': None, 
            'anzahlKommentare': {
                '$sum': '$countComments'
            }
        }
    }, {
        '$project': {
            'anzahlKommentare': 1, 
            '_id': 0
        }
    }
])

for var in result:
    jsonp(var)
```

    {'anzahlKommentare': 267}


Welche Benutzer haben bei Filme mit 100 oder mehr Awards die meisten Kommentare zum gleichen Film erfasst?


```python
result = db.comments.aggregate([
    {
        '$group': {
            '_id': {
                'movie_id': '$movie_id', 
                'email': '$email'
            }, 
            'countComments': {
                '$sum': 1
            }
        }
    }, {
        '$match': {
            'countComments': {
                '$gt': 1
            }
        }
    }, {
        '$sort': {
            'countComments': -1
        }
    }, {
        '$lookup': {
            'from': 'movies', 
            'localField': '_id.movie_id', 
            'foreignField': '_id', 
            'as': 'movie'
        }
    }, {
        '$addFields': {
            'movie': {
                '$first': '$movie'
            }
        }
    }, {
        '$lookup': {
            'from': 'users', 
            'localField': '_id.email', 
            'foreignField': 'email', 
            'as': 'user'
        }
    }, {
        '$addFields': {
            'user': {
                '$first': '$user'
            }
        }
    }, {
        '$match': {
            'movie.awards.wins': {
                '$gt': 99
            }
        }
    }, {
        '$limit': 3
    }, {
        '$project': {
            '_id': 0, 
            'movie.title': 1, 
            'user.name': 1, 
            'countComments': 1
        }
    }
])

for var in result:
    jsonp(var)
```

    {'countComments': 5,
     'movie': {'title': 'American Beauty'},
     'user': {'name': 'Kathryn Sosa'}}
    {'countComments': 5,
     'movie': {'title': 'American Beauty'},
     'user': {'name': 'Daenerys Targaryen'}}
    {'countComments': 4,
     'movie': {'title': 'The Lord of the Rings: The Fellowship of the Ring'},
     'user': {'name': 'Hot Pie'}}


Welche Stadt hat die meisten "Theaters" und in welchem Staat befindet sich die Stadt?


```python
result = db.theaters.aggregate([
    {
        '$addFields': {
            'city': '$location.address.city'
        }
    }, {
        '$group': {
            '_id': '$city', 
            'countTheaters': {
                '$sum': 1
            }
        }
    }, {
        '$sort': {
            'countTheaters': -1
        }
    }, {
        '$lookup': {
            'from': 'theaters', 
            'localField': '_id', 
            'foreignField': 'location.address.city', 
            'as': 'theaters'
        }
    }, {
        '$addFields': {
            'state': {
                '$first': '$theaters.location.address.state'
            }
        }
    }, {
        '$addFields': {
            'city': '$_id'
        }
    }, {
        '$project': {
            'theaters': 0, 
            '_id': 0
        }
    }, {
        '$limit': 10
    }
])

for var in result:
    jsonp(var)
```

    {'city': 'Las Vegas', 'countTheaters': 29, 'state': 'NV'}
    {'city': 'Houston', 'countTheaters': 22, 'state': 'TX'}
    {'city': 'San Antonio', 'countTheaters': 14, 'state': 'TX'}
    {'city': 'Orlando', 'countTheaters': 13, 'state': 'FL'}
    {'city': 'Dallas', 'countTheaters': 12, 'state': 'TX'}
    {'city': 'Los Angeles', 'countTheaters': 12, 'state': 'CA'}
    {'city': 'Atlanta', 'countTheaters': 10, 'state': 'GA'}
    {'city': 'San Francisco', 'countTheaters': 9, 'state': 'CA'}
    {'city': 'Jacksonville', 'countTheaters': 9, 'state': 'FL'}
    {'city': 'Chicago', 'countTheaters': 8, 'state': 'IL'}


In welchem Land wurden die meisten Filme gedreht und wieviel Nominierungen haben diese Filme insgesamt erhalten.


```python
result = db.movies.aggregate([
    {
        '$group': {
            '_id': {
                '$first': '$countries'
            }, 
            'countMovies': {
                '$sum': 1
            }, 
            'countNominations': {
                '$sum': {
                    '$add': [
                        '$awards.nominations', '$awards.wins'
                    ]
                }
            }
        }
    }, {
        '$sort': {
            'countMovies': -1
        }
    }, {
        '$addFields': {
            'country': '$_id'
        }
    }, {
        '$project': {
            '_id': 0
        }
    }, {
        '$limit': 5
    }
])

for var in result:
    jsonp(var)
```

    {'countMovies': 10403, 'countNominations': 96974, 'country': 'USA'}
    {'countMovies': 1980, 'countNominations': 18916, 'country': 'UK'}
    {'countMovies': 1662, 'countNominations': 12400, 'country': 'France'}
    {'countMovies': 894, 'countNominations': 7736, 'country': 'Italy'}
    {'countMovies': 845, 'countNominations': 6424, 'country': 'Canada'}


Welches Genre hat die meisten Nominierungen pro Film? (Filme vom letztem Jahrtausend)


```python
result = db.movies.aggregate([
    {
        '$match': {
            'year': {
                '$lt': 2000
            }
        }
    }, {
        '$group': {
            '_id': {
                '$first': '$genres'
            }, 
            'countMovies': {
                '$sum': 1
            }, 
            'countNominations': {
                '$sum': {
                    '$add': [
                        '$awards.nominations', '$awards.wins'
                    ]
                }
            }
        }
    }, {
        '$addFields': {
            'nominationsPerFilm': {
                '$divide': [
                    '$countNominations', '$countMovies'
                ]
            }
        }
    }, {
        '$addFields': {
            'genre': '$_id'
        }
    }, {
        '$sort': {
            'nominationsPerFilm': -1
        }
    }, {
        '$limit': 3
    }, {
        '$project': {
            '_id': 0
        }
    }
])

for var in result:
    jsonp(var)
```

    {'countMovies': 410,
     'countNominations': 5201,
     'genre': 'Biography',
     'nominationsPerFilm': 12.685365853658537}
    {'countMovies': 6,
     'countNominations': 45,
     'genre': 'History',
     'nominationsPerFilm': 7.5}
    {'countMovies': 48,
     'countNominations': 359,
     'genre': 'Mystery',
     'nominationsPerFilm': 7.479166666666667}


Die besten Filme nach unserem Film-Score. Voraussetzung: Mindestens 10 User Bewertungen


```python
result = db.movies.aggregate([
    {
        '$match': {
            'tomatoes.viewer.numReviews': {
                '$gt': 9
            }, 
            'imdb.votes': {
                '$gt': 9
            }
        }
    }, {
        '$addFields': {
            'customScore': {
                '$add': [
                    {
                        '$multiply': [
                            100, '$tomatoes.viewer.rating'
                        ]
                    }, {
                        '$multiply': [
                            50, '$imdb.rating'
                        ]
                    }, {
                        '$multiply': [
                            1, '$awards.nominations'
                        ]
                    }, {
                        '$multiply': [
                            2, '$awards.wins'
                        ]
                    }
                ]
            }
        }
    }, {
        '$sort': {
            'customScore': -1
        }
    }, {
        '$project': {
            '_id': 0, 
            'title': 1, 
            'customScore': 1, 
            'plot': 1, 
            'year': 1
        }
    }, {
        '$limit': 3
    }
])

for var in result:
    jsonp(var)
```

    {'customScore': 1392.0,
     'plot': 'A medical engineer and an astronaut work together to survive after a '
             'catastrophe destroys their shuttle and leaves them adrift in orbit.',
     'title': 'Gravity',
     'year': 2013}
    {'customScore': 1392.0,
     'plot': 'A medical engineer and an astronaut work together to survive after a '
             'catastrophe destroys their shuttle and leaves them adrift in orbit.',
     'title': 'Gravity',
     'year': 2013}
    {'customScore': 1383.0,
     'plot': 'Illustrated upon the progress of his latest Broadway play, a former '
             "popular actor's struggle to cope with his current life as a wasted "
             'actor is shown.',
     'title': 'Birdman: Or (The Unexpected Virtue of Ignorance)',
     'year': 2014}

