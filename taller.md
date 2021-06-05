## Introducción a DevSecOps

DevSecOps significa integrar la seguridad al desarrollo de las aplicaciones durante todo el proceso. Esta integración no solo requiere las nuevas herramientas, sino también un enfoque organizativo distinto.

En el siguiente workshop vamos a ver como integrar distintas herramientas DevSecOps en nuestros procesos internos. Para ello vamos a usar un stack django que es bastante popular en las startup actuales. Cada lenguaje de programación tiene sus propias herramientas específicas para sus comprobaciones de seguridad.

Lo primero que vamos a hacer es crear un repo de github sin inicializar:

![](https://www.blackdogs.io/wp-content/examples/createrepo.png)

A continuación entraremos en el VPS del taller:

```
ssh devsecops@88.99.187.185		(contraseña taller2021)
```

Crea una carpeta para ti aparte e inicializa un nuevo proyecto de Django:
```
mkdir tunickname && cd tunickname
django-admin startproject devsecops
```

Entramos en la carpeta del proyecto e inicializamos git:
```
cd devsecops/
echo "# django-devsecops" >> README.md
git init
git add *
git config --global user.email "mrk@busy.ninja"
git config --global user.name "busyninja"
git commit -m "first commit"
git remote add origin https://github.com/busyninja/devsecops.git
git push -u origin master
```

#### Primer pre-commit hook

Ahora que tenemos nuestro nuevo proyecto creado vamos a ver como incluir algunas comprobaciones de seguridad e integridad. Para ello empezaremos instalando algunas herramientas direactamente en el equipo de desarrolladores, que harán algunas comprobaciones antes de que se realizen los commits. Git nos proporciona una serie de hooks que podemos usar, por ejemplo tenemos un hook de pre-commit que realizará acciones antes de que se efectue el commit.

Instala pycodestyle y un hook de pre-commit:
```
pip3 install pycodestyle
wget https://www.blackdogs.io/wp-content/examples/pre-commit -O .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

Probamos que funciona como nosotros esperamos, prueba a editar el archivo manage.py y añade 1 linea en blanco al final. Luego prueba a hacer commit de los cambios:

```
vi manage.py 
git add manage.py
git commit -m"Test precomit"
```

Si lo hemos hecho correctamente, deberemos ver como no realiza el commit:

![](https://www.blackdogs.io/wp-content/examples/nodeja.png)


Edita de nuevo el archivo y añade otra linea más que sea un comentario, si ahora hacemos commit veremos que nos deja correctamente:
```
vi manage.py 
git add manage.py
git commit -m"Test precomit"
git push -u origin master
```

![](https://www.blackdogs.io/wp-content/examples/sideja.png)


#### flake8

Ahora que ya entendemos como funciona el pre-commit, vamos a ver como poner checks más completos de una forma ordenada.

Borra el hook anterior e instala la herramienta pre-commit:

```
rm .git/hooks/pre-commit
pip3 install pre-commit
```
Ahora edita un nuevo archivo en la raiz del repo con el nombre `.pre-commit-config.yaml`, que tenga el siguiente contenido:

```
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.4.0  # Use the ref you want to point at
    hooks:
    # comprueba que cumplimos flake8 (linter)
    -   id: flake8
```

Ahora en lugar de pycodestyle usaremos flake8 para comprobar que seguimos el estandard correctamente. Instala los hooks:

```
pre-commit install
```

Realiza ahora la misma prueba de añadir una linea en blanco al final de manage.py, intenta commitear y observa como no te deja. Si lo haces bien deberás ver algo así:

```
vi manage.py 
git add manage.py
git commit -m"Test precomit"
```

![](https://www.blackdogs.io/wp-content/examples/flake8.png)


Añade de nuevo otro comentario debajo y prueva de commitear:

```
vi manage.py 
git add manage.py
git commit -m"Test precomit"
git push -u origin master
```

![](https://www.blackdogs.io/wp-content/examples/flake8sideja.png)


#### archivos grandes

Vamos a seguir poniendo comprobaciones, añade a `.pre-commit-config.yaml` , deberà quedar como el siguiente ejemplo:

```
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.4.0  # Use the ref you want to point at
    hooks:
    # comprueba que cumplimos flake8 (linter)
    -   id: flake8
    # comprueba que no subimos archivos grandes al repo
    -   id: check-added-large-files
```

Crea un nuevo archivo grande de 10mb y comprueba que el precommit bloquea la subida:

```
dd if=/dev/urandom of=archivogrande.bin bs=1M count=10
git add archivogrande.bin
git commit -m"Test precomit"
```

Deberás ver como lo bloquea:

![](https://www.blackdogs.io/wp-content/examples/largeko.png)

Intenta ahora subir un archivo dentro del tamaño permitido:

```
dd if=/dev/urandom of=archivogrande.bin bs=400K count=1
git add archivogrande.bin
git commit -m"Test precomit"
git push -u origin master
```

![](https://www.blackdogs.io/wp-content/examples/largeok.png)

#### ast

Vamos ahora a activar el check de AST que comprobará que el codigo que se sube al repo parsee como python válido, o en otras palabras que no suban código roto al repo. Edita de nuevo el archivo `.pre-commit-config.yaml` y añade los checks AST:

```yaml
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.4.0  # Use the ref you want to point at
    hooks:
    # comprueba que cumplimos flake8 (linter)
    -   id: flake8
    # comprueba que no subimos archivos grandes al repo
    -   id: check-added-large-files
    # comprueba que los archivos se parsean como python válido
    -   id: check-ast
```

Edita ahora el archivo manage.py y añade una definición erronea que rompa el código cómo en el siguiente ejemplo:

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'devsecops.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == '__main__':
    main()

# esto es un comentario

# esto es otro comentario


def broken():
    `
```

Si intentas subir verás que no nos deja:

```
git add manage.py 
git commit -m"Test precomit"
```

![](https://www.blackdogs.io/wp-content/examples/astroto.png)

Arregla ahora la definición:

```python
...
# esto es un comentario

# esto es otro comentario


def broken():
    return True
```

E intentamos subir de nuevo:

```
git add manage.py
git commit -m"Test precomit"
git push -u origin master
```

![](https://www.blackdogs.io/wp-content/examples/astok.png)

#### trailing whitespace

Añadimos ahora el check de trailing whitespaces, que comprueba que no dejemos espacios en blanco al final de alguna línea. Añade a `.pre-commit-config.yaml` la definición:

```yaml
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.4.0  # Use the ref you want to point at
    hooks:
    # comprueba que cumplimos flake8 (linter)
    -   id: flake8
    # comprueba que no subimos archivos grandes al repo
    -   id: check-added-large-files
    # comprueba que los archivos se parsean como python válido
    -   id: check-ast
    # comprueba que no subimos trailing whitespaces
    -   id: trailing-whitespace
```

Edita el archivo manage.py, añade una espacio en blanco al final de cualquier línea que contenga código y haz commit. Si lo has hecho todo bien deberás ver como lo detecta pero también como lo arregla:

![](https://www.blackdogs.io/wp-content/examples/autofixtrailing.png)

Por lo que solo tenemos que commitear de nuevo, pero no habrá cambios.

```
git add manage.py
git commit -m"Test precomit"
```

![](https://www.blackdogs.io/wp-content/examples/fixedtrailing.png)

#### llaves privadas

Añadamos ahora algun check que busque específicamente que no se filtren llaves privadas al repo. Es habitual encontrar llaves ssh o web en los repositorios y esto es muy mala práctica.

Añade la nueva comprovación a `.pre-commit-config.yaml`: 

```yaml
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.4.0  # Use the ref you want to point at
    hooks:
    # comprueba que cumplimos flake8 (linter)
    -   id: flake8
    # comprueba que no subimos archivos grandes al repo
    -   id: check-added-large-files
    # comprueba que los archivos se parsean como python válido
    -   id: check-ast
    # comprueba que no subimos trailing whitespaces
    -   id: trailing-whitespace
    # comprueba que no subimos una llave privada
    -   id: detect-private-key
```

Intenta ahora subir una llave RSA al repo:

```
cp ~/.ssh/id_rsa subolallavealrepo
git add subolallavealrepo 
git commit -m"Test precomit"
```

Deberemos ver como detecta que estamos intentando subir una llave:

![](https://www.blackdogs.io/wp-content/examples/privateko.png)

Arreglamos el problema y subimos los cambios:

```
echo "" > subolallavealrepo 
git add subolallavealrepo 
git commit -m"Test precomit"
git push -u origin master
```

![](https://www.blackdogs.io/wp-content/examples/privateok.png)

#### aws credentials

Es muy habitual también encontrar ID y secrets de AWS en el código. Por lo tanto añadiremos también estas comprobaciones de seguridad.

Añade una nueva comprobación:

```yaml
    # comprueba que no se filtren secretos aws
	-   id: detect-aws-credentials
``` 

Edita el archivo `devsecops/settings.py` y añade el siguiente código:

```python
...

from pathlib import Path
import os

AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent
... 
```

```
git add devsecops/settings.py
git commit -m"Test precomit"
```

Vemos como detecta que se estan filtrando llaves a AWS:

![](https://www.blackdogs.io/wp-content/examples/awsleak.png)

Cambia los valores para que sean recojidos del environment:

```
AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_ACCESS_KEY_KEY')
```

Exportamos las nuevas variables en el entorno y ahora si nos dejará hacer commit y push:

```
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
git add devsecops/settings.py
git commit -m"Test precomit"
git push -u origin master
```

![](https://www.blackdogs.io/wp-content/examples/awskeysok.png)

### jenkins bandit

Ahora vamos a hacer comprobaciones de seguridad desde el Jenkins, que podremos poner en nuestros pipelines de integración continua.

En este caso como usamos django, podemos usar bandit para detectar fallos de seguridad comunes.

Bandit es una herramienta diseñada para buscar fallos de seguridad comunes en código python. Para hacer esto Bandit procesa cada archivo, buildea el AST y ejecuta una serie de plugins. Cuando termina, genera un reporte.

Dirigete ahora a https://jenkinstaller.blackdogs.io/. 

Usuario:	codenoobs
Contraseña:	taller2021

Añade un nuevo proyecto (sin espacios en el nombre), configura el origen de github en source code management, en build steps añade execute shell y escribe lo siguiente:

En español source code management es configurar el origen del código fuente, build step se llama construir ahora y execute shell se llama ejecutar linea de comandos.


```
#!/bin/bash

bandit -r $PWD
```

Si ahora ejecutamos una build veremos que falla porque el debug está activado y hemos subido un secret específico de django.

Arregla ahora el problema:

```
SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = False
```

Sube los cambios realizados al repo.

```
git add devsecops/settings.py
git commit -m"Test precomit"
git push -u origin master
```

Si ejecutas la build de nuevo verás que compila correctamente.

Al poner este paso dentro de un pipeline, cualquier problema que encuentre hará que pare la integración, aportando una capa de seguridad extra.

#### jenkins safety

Safety es una tool que va a comprobar que no tengamos fallos de seguridad en nuestras dependencias de python.

Crea un archivo `requirements.txt` con nuestras dependencias:

```
django==3.2.4
sqlparse==0.2.2
asgiref==3.3.2
jinja2==2.10
```

Sube los cambios al repo:

```
vi requirements.txt 
git add requirements.txt 
git commit -m"Test precomit"
git push -u origin master
```

Ahora añade al jenkins la comprobación adecuada:
```
#!/bin/bash

bandit -r $PWD

cat requirements.txt | safety check --stdin
```

Si ejecutamos la build veremos como encuentra una dependencia vulnerable.

Deberemos arreglar la dependencia vulnerable, en código en producción primero probaremos adecuadamente que el fix no rompe nada, una vez estemos seguros actualizaremos la dependencia:

```
django==3.2.4
sqlparse==0.2.2
asgiref==3.3.2
jinja2==2.11.3
```

```
vi requirements.txt 
git add requirements.txt 
git commit -m"Test precomit"
git push -u origin master
```

Al subir los cambios veremos que sin dependencias vulnerables el jenkins ya da el OK para seguir con nuestro deploy.

Para entornos que ya tengais en productivo vale la pena mirar todas las vulnerabilidades disponibles en un sistema instalado. Esto lo podemos hacer ejecutando el siguiente comando:

```
pip3 freeze | safety check --stdin
```

![](https://www.blackdogs.io/wp-content/examples/safety.png)

