---
title: "Assert Statique"
date: 2014-08-04T15:02:57+02:00
draft: false
---

L’utilisation d’un assert statique peut être utile dans le cas ou une condition doit être testée au moment de la compilation. Par exemple la taille d’un tableau. Le principe est d’utiliser une macro pour définir une expression. Si le test est valide alors l’expression générée l’est aussi, sinon elle résulte en une erreur de compilation.

```c++
#define STATIC_ASSERT(cond)    typedef int ERROR_##__LINE__[(cond) ? 1 : -1]
```

Ici, on utilise `typedef` pour définir un type tableau. Si la condition est valide alors la taille du tableau est positive. Sinon elle est négative ce qui résulte en une erreur de compilation.

Exemple d’utilisation :

```c++
STATIC_ASSERT(sizeof(object1) == sizeof(object2));
```
