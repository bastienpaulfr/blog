---
title: "Protéger les classes contre la copie ou l’assignation en C++"
date: 2014-08-14T15:02:46+02:00
draft: false
---

Pour protéger les classes contre la copie, il suffit de déclarer le constructeurs de copie en `private`. Une macro peut être définie car le principe est le même pour toutes les classes.

```c++
#define DECLARE_NO_COPY_CLASS(classname)  
    private:                              
        classname(const classname&);
#endif
```

De même pour les protéger contre l’assignation, on peut déclarer la surcharge de l’opérateur `=` comme étant `private`.

```c++
#define DECLARE_NO_ASSIGN_CLASS(classname)  
    private:                                
        classname& operator=(const classname&);
#endif
```
