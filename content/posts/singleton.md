---
title: "Singleton"
date: 2014-08-04T15:02:16+02:00
draft: false
---

## Présentation

Le singleton est un patron de conception (design pattern) dont le but est de restreindre l’instanciation  d’une classe à un nombre défini d’objets. Il est utilisé lorsque l’on a besoin que d’un seul objet pour coordonner des actions dans un système. Par exemple, le singleton va être utilisé comme gestionnaire de fenêtre, gestionnaire de systèmes de fichiers ou d’imprimantes. Le singleton va être disponible de manière globale dans un package et sera utilisé comme étant un point d’accès.

Le singleton est une classe qui va s’instancier elle-même. Ses constructeurs seront donc définis comme étant privés. Elle met à disposition une méthode statique pour récupérer son instance. Cette méthode vérifie qu’une seule instance existe ou en crée une si elle n’existe pas encore.

Une attention particulière sera donnée dans les environnements multi-threadés. Si deux thread essaient d’accéder à l’objet, un seul devra l’instancier tandis que le deuxième obtiendra une référence sur l’objet nouvellement créé.

## Implémentation en C++

```c++
template <typename T> class Singleton
{
    public:
        static T* Get() {
            if(null == m_intance) {
                m_instance = new T;
            }
            return m_instance;
        }

    protected:
        Singleton(){};
        virtual ~Singleton(){};

    private :
        static T* m_instance;
};
```

Les classes peuvent ensuite hériter de `Singleton`. Elles seront instanciées par la méthode héritée `Get()`.

```c++
lass Eur : public Singleton<Eur>
{
    friend class Singleton<Eur>;
};

func(){
    Eur* obj = Eur::Get();
};
```

## Multi-Threading

Dans un environnement multi-tâche,  il est important de protéger le code d’instanciation pour éviter d’avoir plusieurs références vers le Singleton lorsque celui-ci est appelé depuis plusieurs threads différents.

La solution peut être comme suit :

```c++
template <typename T> class Singleton
{
    public:
        static T* Get() {
            m_mutex.Lock();
            if(null == m_intance) {
                m_instance = new T;
            }
            m_mutex.unlock();
            return m_instance;
        }

    protected:
        Singleton(){};
        virtual ~Singleton(){};

    private :
        static T* m_instance;
        static Mutex m_mutex;
};
```

Ici `Mutex` est une classe qui initialise le mutex dans son constructeur par défaut et présente les méthodes `Lock()` et `Unlock()` pour locker et délocker le mutex.

Le problème de cette solution est que le mutex est locké à chaque appel de `Get()`. Ceci peut être coûteux en nombre d’instruction en comparaison avec un simple test pour savoir si m_instance existe. D’autant plus que le Singleton n’est instancié qu’une seule fois pendant la durée de vie du programme.

Une proposition d’optimisation serait de tester une première fois si l’instance existe sans locker le mutex puis de tester une seconde fois sous protection dans le cas ou l’instance n’existe pas. Ceci ajoute un test dans le cas ou l’instance n’existe pas, mais enlève tout appel au mutex dans le cas ou elle existe.

```c++
template <typename T> class Singleton
{
    public:
        static T* Get() {
            if(null == instance)
            {
                m_mutex.Lock();
                if(null == m_intance) {
                    m_instance = new T;
                }
                m_mutex.unlock();
            }
            return m_instance;
        }

    protected:
        Singleton(){};
        virtual ~Singleton(){};

    private :
        static T* m_instance;
        static Mutex m_mutex;
};
```

Cette solution est valide car les accès concurrents en lecture ne sont pas pénalisant. En effet, le `Singleton` est le même pour tous les threads.
