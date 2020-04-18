---
title: "Mutex Locker"
date: 2014-08-04T15:02:32+02:00
draft: false
---

## Préambule

Le mutex locker fait partie du concept de [RAII](http://fr.wikipedia.org/wiki/RAII). Cette acronyme vient de l’anglais Resource Acquisition Is Initialization. C’est une technique de programmation qui permet de s’assurer que la ressource acquise sera bien libérée à la fin de la vie d’un objet.

## Principe

Appliqué au mutex, ce principe consiste à bloquer un mutex lors de la création d’un objet et de le libérer lors de sa destruction. On peut imaginer la création d’un objet spécifique : le `MutexLocker`.  Il faudrait qu’il puisse prendre un mutex lors de sa création et le libérer lors de sa destruction. Son constructeur va donc prendre le mutex et son destructeur le détruire.

```c++
class MutexLocker {
    public :
        MutexLocker(Mutex& mutex): m_mutex(mutex) {
            m_mutex.Lock();
        }
        virtual ~MutexLocker(){
            m_mutex.Unlock();
        }

    private:
        Mutex& m_mutex;
};
```

Son utilisation procède comme suit :

```c++
{
    Mutex mutex;
    MutexLocker locker(mutex);
    /* Maintenant le mutex est pris. Le code critique peut être exécuté. */
}
```

## Conclusion

Son utilisation est justifié quand des exceptions peuvent arriver à tout moment. A ce moment, le mutex est assuré d’être libéré à la fin de la portée du code lorsque le locker est détruit. D’une manière générale, il est bon de l’utiliser tout le temps pour éviter de tomber dans des cas d’erreur difficiles à détecter.
