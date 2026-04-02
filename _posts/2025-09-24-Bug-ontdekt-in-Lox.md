---
layout: post
title: "Bug ontdekt in Lox: `this` variabele actief buiten klassemethodes"
date: 2025-09-24 14:00:00 +0200
categories: Bug Report
published: false
---
## 🐞 Bug ontdekt in Lox: `this` variabele actief buiten klassemethodes

### 🧠 Achtergrond

Lox is een eenvoudige objectgeoriënteerde scripttaal, geïmplementeerd in het boek _Crafting Interpreters_ van Bob Nystrom.

Tijdens het bestuderen van de taal stuitte ik op iets vreemds:  
Het gebruik van `this` **buiten** een klassemethode gaf **geen** runtime-fout, terwijl je dat wél zou verwachten.

Maar `this` mag toch alleen binnen klassemethodes gebruikt worden?

---

### ❗ Het Probleem

Kijk eens naar deze eenvoudige code:

`print this;`

Deze regel staat in de **globale scope** — dus niet binnen een klasse of methode.  
Normaal gesproken zou de interpreter een foutmelding moeten geven zoals:

`Can't use 'this' outside of a class.`

Maar in werkelijkheid gebeurt er niets.  
Erger nog: `this` gedraagt zich als een geldige lokale variabele!

---

### 🔁 Reproductie van het Probleem

Je kunt de bug als volgt reproduceren:

1. **Schakel de runtime-check uit** die voorkomt dat `this` buiten een klasse wordt gebruikt. Zoek in de compiler of resolver naar:
 ```c   
    `if (currentClass == NULL) {     
	    error("Can't use 'this' outside of a class.");     
	    return; 
	}`
```
1. Draai nu Lox-code waarin `this` **buiten** een klasse of methode wordt gebruikt, bijvoorbeeld:
    
    `print this;`
    
2. **Verwacht gedrag:** Een runtime-fout die aangeeft dat `this` onjuist gebruikt wordt.
    
3. **Huidig gedrag:**  
    Geen foutmelding. `this` gedraagt zich als een normale lokale variabele die is opgeslagen in **slot 0** van het activatierecord (de stackruimte voor lokale variabelen).
    

Dit is een **stille fout** — en die zijn vaak het gevaarlijkst.

---

### 🔍 Oorzaakanalyse

In de compiler van Lox wordt tijdens het compileren van een functie de volgende logica toegepast:
```c   
`if (type != TYPE_FUNCTION) {     
	local->name.start = "this";     
	local->name.length = 4; 
} else {     
	local->name.start = "";     
	local->name.length = 0; 
}`
```
Hierbij wordt `"this"` standaard toegevoegd aan **slot 0**, zelfs als de functie geen methode is.

Het gevolg? `this` verschijnt in de lokale variabelentabel en wordt door de runtime als geldig beschouwd — ook al is er geen klasse in zicht.

---

### ✅ De Oplossing

Om te voorkomen dat `this` wordt toegevoegd aan functies die **geen** methodes zijn, moeten we controleren of we ons daadwerkelijk in een klasse bevinden:
```c   
`if (currentClass != NULL && type != TYPE_FUNCTION) {     
	local->name.start = "this";     
	local->name.length = 4; 
} else {     
	local->name.start = "";     
	local->name.length = 0; 
}`
```
Dankzij deze aanpassing wordt `"this"` alleen als lokale variabele geregistreerd:

- Als we binnen een klasse zitten (`currentClass != NULL`)
    
- En als we geen gewone functie aan het compileren zijn
    

Vanaf nu zal elk verkeerd gebruik van `this` buiten een klasse correct resulteren in een runtime-fout.

---

### 🧪 GitHub Issue en Pull Request

Ik heb deze bug gemeld in de [officiële GitHub-repository van Crafting Interpreters](https://github.com/munificent/craftinginterpreters) en een Pull Request ingediend met de oplossing:

- 📌 **Issue:** [Suggestion: Avoid Registering this in Slot 0 for Non-Method Functions #1203](https://github.com/munificent/craftinginterpreters/issues/1203)
    
- 🔧 **Pull Request:** [Suggestion: Only assign `this` to slot 0 if `currentClass` is not null](https://github.com/munificent/craftinginterpreters/pull/1202)
    

Alle bestaande tests slagen, en deze fix zorgt voor correcter en veiliger gedrag van `this`.

---

### 🧭 Conclusie

Wat begon als een kleine observatie leidde tot een interessante bugfix.

Door dieper in de compiler te duiken en het gedrag van scopes en variabelen te analyseren, leerde ik:

- Hoe kleine aannames grote gevolgen kunnen hebben
    
- Waarom compiler-logica scherp afgebakend moet zijn
    
- Hoe belangrijk runtime-validatie is voor taalveiligheid
    

✨ Het mooiste van alles? Ik mocht bijdragen aan een open-source project dat ik zelf gebruik om te leren.

---

### 📚 Referenties

- 📘[Crafting Interpreters (boek)](https://craftinginterpreters.com/)
    
- 💻 [GitHub Repository](https://github.com/munificent/craftinginterpreters)
    
- 🐛 [Bug Report op GitHub](https://github.com/munificent/craftinginterpreters/issues/1203)
    
- 🔧 [Pull Request met de fix](https://github.com/munificent/craftinginterpreters/pull/1202)
    

---
