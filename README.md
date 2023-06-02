#front #js

Récemment, mon équipe a dû faire communiquer deux iframes sans avoir la main sur la root page qui les invoquées. Il nous était donc impossible de les faire communiquer via la root page avec `postMessage`. Nos iframes ayant le même domaine, nous pouvons les faire communiquer par le localStorage.

En effet, lorsqu'un document HTML modifie une entrée du localStorage un event est envoyé aux autres documents partageant le même localStorage. Il est bien important de noter que le document ayant déclenché l'event ne recevra aucun event.


## Cas pratique : initialisation

Cet exemple utilisera un [dépôt git](https://github.com/LesageYann/storage-event-example) en support.

Nous allons créer la page `root.html` qui illustrera la page sur laquelle nous n'avons pas la main :
```html
<!DOCTYPE html>  
<html lang="en">  
<head>  
    <title>Root page</title>  
</head>  
<body>  
<h1>Root page</h1>  
<iframe src="emitter.html" width="400" height="500"></iframe>  
<iframe src="receiver.html" width="400" height="500"></iframe>  
  
</body>  
</html>
```

Aucun évènement, aucun Javascript ne sera ajouté ici pour simuler l'absence de maitrise sur l'hôte de nos pages.

Nous allons ensuite faire un fichier `emitter.html` qui enverra un évènement qui devra être reçu par le fichier `receiver.html`.

```html 
<!DOCTYPE html>  
<html lang="en">  
<head>  
    <title>Emitter page</title>  
</head>  
<body>  
<h1>Emitter page</h1>  
  
<button onclick="send('PS1')">PS1 is the better console</button>  
<button onclick="send('N64')">N64 has the better games</button>  
<button onclick="send('PC')">the pc survived with a large catalog of games</button>  
  
<script>  
  
    function send(name) {  
        // TODO  
    }  
  
</script>  
  
</body>  
</html>
```

Passons au `receiver.html`. Sa tâche sera simplement de recevoir un event et d'afficher le nombre de votes qu'a reçu la plateforme de jeux
```html
<!DOCTYPE html>  
<html lang="en">  
<head>  
    <title>Receiver page</title>  
</head>  
<body>  
<h1>Receiver page</h1>  
    
<div> the ps1 is best gaming platform - <span id="PS1">0</span> votes</div>  
<div> the n64 is best gaming platform - <span id="N64">0</span> votes</div>  
<div> the pc is best gaming platform - <span id="PC">0</span> votes</div>  
  
</body>  
</html>
```


## Ecouter un event

Git checkpoint : vous pouvez sauter au tag: #initialization

Nous allons ajouter le code suivant au `receiver.html` :
```html 
<script>  
    let counters = new Map([["PS1", 0],["N64", 0], ["PC", 0]]);  
    window.addEventListener("storage", () => {  
        let name = JSON.parse(window.localStorage.getItem("best-console"))?.name;  
        let count = counters.get(name) + 1;  
        counters.set(name, count);  
        document.getElementById(name).innerText = count;  
        window.localStorage.setItem("best-console", JSON.stringify({name, validatedAt: new Date() }));  
    });  
</script>
``` 

Nous avons donc ajouté un listener sur windows "storage". Aucun paramètre, nous devons uniquement connaitre quelle valeur récupérer dans notre storage. Nous modifions les span avec le nombre de votes pour avoir un résultat visuel.

## Envoyer un event

Git checkpoint : vous pouvez sauter au tag: #listen-events

Pour envoyer un évènement dans le storage, nous allons utiliser la fonction `localStorage.setItem`. Aucun évènement n'est directement déclenché. A chaque fois que les valeurs du local storage change dans une autre fenêtre, un event est emit. Voici le nouveau script de la page `emitter.html`
```html
<script>  
  
    function send(name) {  
        localStorage.setItem('best-console', JSON.stringify({name}));  
    }  
  
</script>
```


## Forcer un évènement à être rejoué

Git checkpoint : vous pouvez sauter au tag: #working-events

Horreur ! Malheur !  Quand nous cliquons plusieurs sur le même bouton, le compteur de vote n'est incrémenté que d'un !  La raison est simple et nous l'avons déjà vu plus haut : l'event n'est déclenché qu'une fois que l'état du local storage change. Hors si nous appuyons deux fois sur "pc", la deuxième fois, la valeur reste identique.

Pour jouer un évènement, l'astuce consiste donc à associer un compteur ou un id. Voici un exemple avec un compteur :

`emitter.html` :
```html
<script>  
  
    let key = 0;  
  
    function send(name) {  
        key = key + 1;  
        localStorage.setItem('best-console', JSON.stringify({name, key}));  
    }  
  
</script>
```


## Distinguer les clefs mises à jour

Git checkpoint : vous pouvez sauter au tag: #repeated-events

Bien imaginons que nous souhaitons réinitialiser les votes,  nous allons ajouter une clef avec un compteur pour ça :

`emitter.html`
```html
<button onclick="reset()">reset</button>

<script>  
    let key = 0;  
  
    function send(name) {[...]}  

	function reset() {
		localStorage.setItem('reset-vote', key)
	}
</script>
``` 

`receiver.html` :
```html
<script>  
    let counters = new Map([["PS1", 0],["N64", 0], ["PC", 0]]);  
  
    function vote() {  
        let name = JSON.parse(window.localStorage.getItem("best-console"))?.name;  
        let count = counters.get(name) + 1;  
        counters.set(name, count);  
        document.getElementById(name).innerText = count;    
        window.localStorage.setItem("best-console", JSON.stringify({name, validatedAt: new Date() }));  
    }  
  
    function reset() {  
        counters = new Map([["PS1", 0],["N64", 0], ["PC", 0]]);  
        ["PS1", "N64", "PC"].forEach(name => document.getElementById(name).innerText = "0");  
    }  
  
    window.addEventListener("storage", (event) => {  
        switch (event.key) {  
            case "best-console":  
                vote();  
                break;            
            case "reset-vote":  
                reset()  
        }  
    });  
</script>
```


## Attention aux boucles

Git checkpoint : vous pouvez sauter au tag: #multiple-events

Nous avons un système qui marche, mais pourquoi ne pas ajouter un logger pour voir un peu mieux ce qui se passe. Ajoutons donc ces codes :

`root.html`
```html
<iframe src="logger.html" width="400" height="500"></iframe> 
```

`logger.html`
```html
<!DOCTYPE html>  
<html lang="en">  
<head>  
    <title>Logger page</title>  
</head>  
<body>  
<h1>Logger page</h1>  
    
<div id="displayer"></div>  
<script>   
  window.addEventListener("storage", (event) => {    
    const newP = document.createElement("p");  
    let newContent  
    switch (event.key) {  
        case "best-console":  
            const name = JSON.parse(window.localStorage.getItem("best-console"))?.name;  
            newContent = document.createTextNode("new vote for " + name);  
            window.localStorage.setItem("best-console", JSON.stringify({name, loggedAt: new Date() }));  
            break;      
        case "reset-vote":  
            newContent = document.createTextNode("Votes are reset !");  
            window.localStorage.setItem("reset-vote", JSON.stringify({loggedAt: new Date() }));  
        }  
    newP.appendChild(newContent);  
    document.getElementById("displayer").appendChild(newP);
  });  
</script> 
</body>  
</html>
```

Git checkpoint : vous pouvez sauter au tag: #endless-loops


Lançons la page pour voir le résultat. En lançant un nouveau code, vous pourrez constater une magnifique boucle infinie. La raison est assez simple : à chaque nouvelle modification de l'entrée par une page, un nouvel évènement est généré pour les deux autres pages. Si nous n'avions pas ce problème quand la page `receiver.html` modifiait l'évent, c'est qu'il y avait un évenement généré mais écouté par personne. Les solutions possibles sont :
- distinguer les évènements
  ou
- retenir les keys de chaque vote pour éviter de rejouer les events
  ou
- ne pas tenir compte des évènements sans {key}

Implémentons la troisième solution :

remplaçons les lignes dans `receiver.html`  et `logger.html`

```js
const name = JSON.parse(window.localStorage.getItem("best-console"))?.name; 
```
par :
```js 
const content = JSON.parse(window.localStorage.getItem("best-console"))
if(!content?.key) {
	return 
}
const name = content?.name;
```

## Conclusions

Git checkpoint : vous pouvez sauter au tag: #final

Comme vous avez pu le constater, la mise en place de ce système mérite quelques précautions et est [utilisable par 96% des utilisateurs ](https://caniuse.com/mdn-api_window_storage_event) mais il évite de recourir au réseau et à un serveur.  Ces derniers avantages ne sont pas à négliger à une époque où on veut toujours tout plus vite pour moins cher et moins d'impact sur la planète !


## bibliographie

- [iframe - FR](https://developer.mozilla.org/fr/docs/Web/HTML/Element/iframe)
- [postMessage - FR](https://developer.mozilla.org/fr/docs/Web/API/Window/postMessage)
- [storage event](https://developer.mozilla.org/fr/docs/Web/API/Window/storage_event)
- [can I use](https://caniuse.com/mdn-api_window_storage_event)
