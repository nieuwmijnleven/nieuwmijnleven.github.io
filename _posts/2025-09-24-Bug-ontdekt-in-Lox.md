---
layout: post
title: "Bug ontdekt in Lox: `this` variabele actief buiten klassemethodes"
date: 2025-09-24 14:00:00 +0200
categories: Bug Report
---
## 🐞 Bug ontdekt in Lox: `this` variabele actief buiten klassemethodes

### Achtergrond

[Lox](https://craftinginterpreters.com/) is een eenvoudige objectgeoriënteerde scripttaal die wordt geïmplementeerd in het boek [Crafting Interpreters](https://craftinginterpreters.com/).  
Tijdens het bestuderen van de taal kwam ik iets merkwaardigs tegen:  
**Het gebruik van `this` buiten een klasse of methode gaf geen runtime-fout**, zoals je wel zou verwachten.

> Maar `this` mag toch alleen binnen klassemethodes worden gebruikt?

---

### Het probleem

Bekijk dit voorbeeld:

```lox
print this;
```

Deze regel staat in de globale scope — dus buiten een klasse of methode.  
**Normaal gesproken zou dit een fout moeten opleveren.** Bijvoorbeeld tijdens het resolven of op runtime.

Maar als je de check in de resolver uitschakelt, gebeurt er iets vreemds:

> ⚠️ De code geeft geen foutmelding, en `this` lijkt gewoon te werken als een lokale variabele!

---

### Analyse van de oorzaak

In de compiler van Lox, wordt bij het compileren van een functie het volgende stuk code uitgevoerd:

```c
if (type != TYPE_FUNCTION) {
  local->name.start = "this";
  local->name.length = 4;
} else {
  local->name.start = "";
  local->name.length = 0;
}
```

Hier wordt `"this"` standaard toegevoegd aan **slot 0 van het lokale variabelenframe** —  
zelfs als de functie **geen methode is van een klasse**.

Daarom wordt `this` geïnterpreteerd als een normale lokale variabele, ook buiten een klasse.  
**De runtime denkt dan dat `this` geldig is, omdat het correct in het geheugen is geplaatst.**

---

### De oplossing

We kunnen dit probleem oplossen door alleen `"this"` toe te voegen als we zeker weten dat we ons in een klassemethode bevinden.  
Dus: we controleren ook of `currentClass` niet `NULL` is.

Aangepaste code:

```c
if (currentClass != NULL && type != TYPE_FUNCTION) {
  local->name.start = "this";
  local->name.length = 4;
} else {
  local->name.start = "";
  local->name.length = 0;
}
```

Met deze wijziging zal `this` alleen als lokale variabele worden geregistreerd in methodes van een klasse.  
**Gebruik van `this` buiten een klasse resulteert dan weer correct in een runtime-fout.**

---

### GitHub Issue en PR

Ik heb deze bug gemeld op de officiële GitHub-repository van Crafting Interpreters,  
en een Pull Request ingediend met de bijbehorende fix:

- 📌 Issue: [# Bug Report: `this` Variable Registered in Slot 0 Even Outside of Class Methods #1201](https://github.com/munificent/craftinginterpreters/issues/1201)
    
- 🔧 Pull Request: [# Fix: Only assign 'this' to slot 0 if currentClass is not null #1202](https://github.com/munificent/craftinginterpreters/pull/1202)
    

> Het was erg leerzaam én leuk om een bug te vinden, analyseren en zelf op te lossen in een project waar ik van leer.

---

### Conclusie

Een kleine observatie leidde tot een interessante ontdekking.  
Door te experimenteren en de compiler beter te begrijpen, vond ik een bug die anders onopgemerkt had kunnen blijven.

Dit soort ervaringen maakt het werken met taalontwerp extra boeiend.  
En nog mooier: ik kon bijdragen aan een open source project waar ik zelf veel van geleerd heb.

> Dus: blijf nieuwsgierig, blijf experimenteren — wie weet wat je zelf ontdekt!

---

### 📚 Referenties

- 📘 [Crafting Interpreters (Boek)](https://craftinginterpreters.com/)
    
- 💻 [GitHub Repository](https://github.com/munificent/craftinginterpreters)
    
- 🐛 [Mijn Bug Report op GitHub](https://github.com/munificent/craftinginterpreters/issues/nieuwmijnleven)
    

---

