# Introduction aux extensions Python avec CFFI

Vous avez réalisé une analyse de votre code et vous avez un bottleneck ?
Vous souhaitez utiliser une bibliothèque bas niveau (C/C++/Rust) ?

Pas de problème, dans cet article, je vais vous expliquer les différentes solutions et pénétrer en profondeur dans la plus charmante d'entre elle, son petit nom: CFFI.

Sam&Max me faisant l'honneur d'accepter mon article, je vais suivre la guideline du site avec un langage détendu et beaucoup d'exemples.


## Qu'est ce qu'une extension Python ?

Guido, pendant l'acte créateur, n'a pas oublié une chose importante: les extensions Python !
Une extension Python est un module compilé pouvant être importé dans votre code Python.
Cette fonctionnalité est très puissante: cela vous permet d'utiliser un langage bas niveau (et toute ses capacités) pour créer un module Python.
Vous utilisez très probablement des extensions Python dans vos projets sans le savoir.
Par exemple, si vous souhaitez embarqué la bibliothèque de calcul physique [Bullet](https://github.com/bulletphysics/bullet3), vous pouvez le faire au sein d'une extension Python.


Il y a deux points à différencier:

- Importer une bibliotèque tierce
- Améliorer les performances de son code

En y réfléchissant bien, créer un code performant revient à créer un bibliotèque tierce et l'importer au sein de l'interpréteur.


### Ca laisse rêveur, alors comment fait on ?

Plusieurs solutions:

1. L'API C de CPython. Y'en a qu'ont essayé, ils ont eu des problèmes...
   En utilisant cette méthode, vous aurez accès à tout l'interpréteur Python et vous pourrez tout faire... mais à quel prix ?
   Je ne vous recommande pas cette approche, je me suis cassé les dents pendant 4 mois dessus avec un succès mitigé.
2. Cython est une bonne solution mais plus orienté sur l'optimisation de code.
3. CFFI: le saint graal, aleluya!


## CFFI: Première mise en bouche

CFFI va vous permettre de créér des extensions Python mais pas que...
Tout comme le module `ctypes`, il va permettre d'importer une bibliothèque dynamique au runtime.
Si vous ne savez pas ce qu'est une bibliothèque, c'est par [ici](https://fr.wikipedia.org/wiki/Biblioth%C3%A8que_logicielle).

CFFI va donc vous permettre:

- D'importer une bibliotèque dynamique au runtime comme `ctypes` mais avec une meilleure API -> Mode ABI -> Pas de compilation
- De réaliser une extension Python compilée comme `cython` ou comme avec l'API C de CPYTHON  -> Mode API -> Phase de compilation

Par rapport à `ctypes`, CFFI apporte une API pythonic et légère, l'API de `ctypes` étant lourde.
Par rapport à l'API C de CPython... Ha non! Je n'en parle même pas de celle-là!


### J'ai dit qu'il y aurait beaucoup d'exemples, alors c'est parti !

D'abord, on installe le bouzin, il y a une dépendance système avec `libffi`, sur Ubuntu: 

```
sudo apt-get install libffi-dev python3-dev
```

Sur Windows, `libffi` est embarquée dans CPython donc pas de soucis.
Ensuite, on conserve les bonnes habitudes avec le classique:

```
pip install cffi
```

Je vous conseille d'utiliser un `virtualenv` mais ce n'est pas le sujet!

### Les trois modes

Il y a trois moyens d'utiliser CFFI, comprenez bien cela car c'est la partie tricky:

1. Le mode ABI/Inline
2. Le mode API/Out-of-line
3. Le mode ABI/Out-of-line

On a déjà évoqué les modes ABI et API, mais je n'ai pas encore parlé de Inline et Out-of-line.
CFFI utilise une phase de "compilation" pour parser les header C. Ce n'est pas une vrai compilation mais une phase de traitement qui peut être lourde.
Le mode Inline signifie que ce traitement va être effectué à l'import du module alors que Out-of-line met en cache ce traitement à l'installation du module.

Evidemment, le mode API/Inline ne peut pas exister puisque le mode API impose une phase de "vrai" compilation.

#### Le mode ABI/Inline

```
# On commence par import le module cffi qui contient la classe de base FFI
from cffi import FFI

# 1 - On instancie l'object FFI, cet objet est la base de cffi
ffi = FFI()

# 2 - On appelle la méthode cdef.
# Cette méthode attend en paramètre un header C, c'est à dire
# les déclarations des fonctions C qui seront utilisées par la suite.
# CFFI ne connaîtra que ce qui a été déclaré dans le cdef.
# La puissance de CFFI réside dans cette fonction, à partir d'un header C,
# il va automatiquement créer un wrapper léger.
# A noter: le code dans cdef ne doit pas contenir de directive pré-processeur.
# Ici, on déclare la fonction printf appartenant au namespace C
ffi.cdef("""
    int printf(const char *format, ...);
""")

# 3 - On charge la bibliothèque dynamique
# dlopen va charger la biliothèque dynamique et la stocker dans la variable nommée cvar.
# L'argument passé est None, cela demande à cffi de charger le namespace C.
# On peut ici spécifier un fichier .so (Linux) ou .dll (Windows).
# Seul ce qui a été déclaré dans cdef sera accessible dans cvar.
cvar = ffi.dlopen(None)
```

Comme vous pouvez le voir dans ce bout de code, CFFI est très simple d'utilisation, il suffit de copier le header C pour avoir accès aux fonctions de la bibliothèque.
*A noter: si les déclarations dans le cdef ne correspondent pas aux déclarations présentes dans la bibliothèque (au niveau ABI), vous obtiendrez une erreur de segmentation.*


#### Le mode API/Out-of-line

Pour bien comprendre ce mode, nous allons implémenter la fonction factorielle.

```
# Comme pour le mode ABI, FFI est la classe principale
from cffi import FFI

# Par convention, en mode API, on appelle l'instance ffibuilder car le compilateur va être appelé
ffibuilder = FFI()

# En mode API, on utilise pas dlopen, mais la fonction set_source.
# Le premier argument est le nom du fichier C à générer, le 2e est le code source.
# Ce code source va être passé au compilateur, il peut donc contenir des directives pré-processeur.
# Dans l'exemple, je passe directement le code source mais en général, on va plutôt ouvrir le fichier avec open().
ffibuilder.set_source("_exemple", """
    long factorielle(int n) {
        long r = n;
        while(n > 1) {
            n -= 1;
            r *= n;
        }
        return r;
    }
""")

# Comme pour le mode ABI, on déclare notre fonction avec la méthode cdef.
ffibuilder.cdef("""
    long factorielle(int);
""")


# Enfin, on va appeler la méthode compile() qui génère l'extension en 2 étapes:
# 1 - Génération d'un fichier C contenant la magie CFFI et notre code C
# 2 - Compilation de ce fichier C en extension Python
if __name__ == "__main__":
    ffibuilder.compile(verbose=True)
```

Après éxécution du script, voici ce que l'on voit dans le terminal:

```
generating ./_exemple.c  -> Étape 1: Génération du fichier C
the current directory is '/home/realitix/test'
running build_ext
building '_exemple' extension  -> Étape 2: Génération de l'extension
x86_64-linux-gnu-gcc -pthread -DNDEBUG -g -fwrapv -O2 -Wall -Wstrict-prototypes -g -fdebug-prefix-map=/build/python3.6-sXpGnM/python3.6-3.6.3=. -specs=/usr/share/dpkg/no-pie-compile.specs -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 -fPIC -I/home/realitix/venv/py36/include -I/usr/include/python3.6m -c _exemple.c -o ./_exemple.o
x86_64-linux-gnu-gcc -pthread -shared -Wl,-O1 -Wl,-Bsymbolic-functions -Wl,-Bsymbolic-functions -specs=/usr/share/dpkg/no-pie-link.specs -Wl,-z,relro -Wl,-Bsymbolic-functions -specs=/usr/share/dpkg/no-pie-link.specs -Wl,-z,relro -g -fdebug-prefix-map=/build/python3.6-sXpGnM/python3.6-3.6.3=. -specs=/usr/share/dpkg/no-pie-compile.specs -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 ./_exemple.o -o ./_exemple.cpython-36m-x86_64-linux-gnu.so
```

Et vous pouvez trouver l'extension Python `_exemple.cpython-36m-x86_64-linux-gnu.so`.
Étudions le module généré, dans un interpréteur Python:

```
>>> from _exemple import ffi, lib
>>> dir(ffi)
['CData', 'CType', 'NULL', 'RTLD_DEEPBIND', 'RTLD_GLOBAL', 'RTLD_LAZY', 'RTLD_LOCAL', 'RTLD_NODELETE', 'RTLD_NOLOAD', 'RTLD_NOW', '__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'addressof', 'alignof', 'buffer', 'callback', 'cast', 'def_extern', 'dlclose', 'dlopen', 'errno', 'error', 'from_buffer', 'from_handle', 'gc', 'getctype', 'init_once', 'integer_const', 'list_types', 'memmove', 'new', 'new_allocator', 'new_handle', 'offsetof', 'sizeof', 'string', 'typeof', 'unpack']
>>> dir(lib)
['factorielle']
```

Les modules générés par CFFI contiennent 2 objets ffi et lib.
 - ffi: Les fonctions de l'API CFFI ainsi que les typedef et structs
 - lib: Toutes nos fonctions C, ici, il n'y a que factorielle

**Ca vous dit d'utiliser notre extension avec un petit test de performance ? Allons y !**

```
import time
from contextlib import contextmanager

# On import lib qui contient notre fonction factorielle
from _exemple import lib

# On créé l'équivalent de notre fonction C en Python
def py_factorielle(n):
    r = n
    while n > 1:
        n -= 1
        r *= n
    return r

# Un petit contextmanager pour mesurer le temps
@contextmanager
def mesure():
    try:
        debut = time.time()
        yield
    finally:
        fin = time.time() - debut
        print(f'Temps écoulé: {fin}')

def test():
    # On va réaliser un factorielle 25 un million de fois
    loop = 1000000
    rec = 25
    # Version Python
    with mesure():
        for _ in range(loop):
            r = py_factorielle(rec)
    # Version CFFI
    with mesure():
        for _ in range(loop):
            r = lib.factorielle(rec)

if __name__ == '__main__':
    test()
```

Le résultat sur ma machine:

```
Temps écoulé: 1.9101519584655762
Temps écoulé: 0.13172173500061035
```

La version CFFI est 14 fois plus rapide, pas mal !
CFFI permet de faire vraiment beaucoup de choses simplement, je ne vous ai montré que la surface afin de vous donner envie d'aller plus loin.


## Ou trouver des ressources

- La doc de CFFI est vraiment bien, un bon readthedocs classique: [cffi.readthedocs.io](https://cffi.readthedocs.io)
- Une de mes présentations à PyConAU, la version EuroPython est moins bonne: [ICI](https://www.youtube.com/watch?v=P6qvhoQGaSk)
- Mes projets CFFI:
    1 - vulkan: Mode ABI [ICI](https://github.com/realitix/vulkan)
    2 - Pour les curieux, la version en utilisant l'API C de CPython [ICI](https://github.com/realitix/cvulkan)
    3 - vulk-bare: Mode API, un module très simple [ICI](https://github.com/realitix/vulk-bare)
    4 - PyVma: Celui-là est très intéressant, mode API qui étend le module vulkan qui en mode ABI, c'est un très bon exemple


A savoir que CFFI a été créé par Armin Rigo and Maciej Fijalkowski, les deux créateurs de Pypy.
Toutes les extensions créées avec CFFI sont compatibles avec Pypy !


## Conclusion

J'espère que cette introduction vous a plu. Si les retours sont bons, je pourrai m'atteler à un tuto plus conséquent.
Vive Python !

Si vous avez des remarques, n'hésitez pas à me le faire savoir: @realitix sur Twitter
