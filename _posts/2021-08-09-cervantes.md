---
title: La letra mas usada por Cervantes.
author: Juan Jose Otalora Gonzalez
date: 2021-08-09
category: Jekyll
layout: post
---

 ...o cómo usar Python y Beautiful Soup para respaldar los datos de wikipedia sobre la frecuencia de aparición de letras del castellano en El Quijote.


## Contexto.

Buscando algo de inspiración para echar un vistazo al _web scrapping_, recordé que la letra que más veces aparece repetida en el lenguaje castellano es la "e", con un ~14% de las mismas. Y pensé que qué mejor manera de ver si es así y al mismo tiempo scrapear un documento con las suficientes letras que hacerlo con, bueno, [El Quijote](http://www.cervantesvirtual.com/obra-visor/el-ingenioso-hidalgo-don-quijote-de-la-mancha--0/html/)

Así pues, en wikipedia comprobé para mi sorpresa que no sólo se encuentra un articulo sobre la aparición de las letras del castellano, sino que hace con dististas fuentes para tener un mejor contraste dependiendo del texto: _La RAE_, _La Celestina_ y, por suerte, [_El Quijote!_](https://es.wikipedia.org/wiki/Frecuencia_de_aparici%C3%B3n_de_letras#Ejemplo_concreto:_el_Quijote)

Por tanto, ya con todos los ingredientes preparados... del plato a la boca que se enfría la sopa.
## Pasos a seguir

Una vez que sabemos nuestro objetivo, me propuse realizar los siguientes pasos:

1. Conseguir pasar El Quijote de su formato online a txt parseado.
2. Una vez tenemos el texto, modificarlo para formular una forma de contarlo.
3. Realizar la cuenta sobre las letras, transformando los acentos a su carácter sin acento correspondiente.

Por tanto, lo que hice fué hacer tres scripts por separado para hacer cada una de las tareas. Con esto presente, vamos pues al primer paso.

### De la web a txt

Ésta es mi primera iteración con BS4 e inspeccionando HTMLs para ver un poco la distribución de sus páginas, pero nos sirve para dar un primer vistazo a cómo funciona.

Beautiful Soup es un [módulo de python](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) que nos permite, junto con _requests_ obtener una página HTML y poder quedarnos con los datos que nos interese, tanto si es como avanzar a otras ramas de la misma web, como si es para recolectar algún tipo de datos, como es nuestro caso.

El código de ésta parte es el siguiente:

```python
from bs4 import BeautifulSoup
import requests, time
pages = 23
getOut = False
#Abrimos un archivo de texto para almacenar los parrafos.
with open("Quijote.txt","w") as quijote:
    for index in range(2,pages):
        getOut = False
        url = "http://www.cervantesvirtual.com/obra-visor/el-ingenioso-hidalgo-don-quijote-de-la-mancha--0/html/fef04e52-82b1-11df-acc7-002185ce6064_{}.html".format(index)
        webpage_response = requests.get(url)
        webpage = webpage_response.content
        soup = BeautifulSoup(webpage,"html.parser")

        #Procedemos a ir por nuestro objeto soup parrafo a parrafo, guardandolo en el txt.
        for paragraph in soup.find_all("p"):
            for string in paragraph.stripped_strings:
                if not ("One fine body" in string):
                    quijote.write(string)
                else:
                    getOut = True
                    break
            if getOut == True:
                print("Parte terminada!")
                break

        print("Vamos a esperar a hacer otra request 1s para no hacer demasiadas peticiones al sitio.")
        print("Iteracion numero: {index}".format(index=index))
        time.sleep(1)

```

Vamos a ir viendo paso por paso que hemos hecho:

#### Obtención de las páginas necesarias.

Lo primero que hacemos es decirle a python que nos cree un nuevo archivo de texto, de nombre ```Quijote.txt``` donde podamos guardar el texto que vamos a irle solicitando. A su vez, si accedemos a la web de _cervantesvitual_ y miramos los capítulos del Hidalgo de la Mancha, podemos observar como el html que cambia entre ..**2.html** para el primer capítulo y ..**22.html** para el último.

Tenemos una forma de realizar 21 peticiones distintas para cada una de las páginas, gracias al módulo ```request```. Podemos seguidamente usar el objeto ```soup``` que hemos creado con ```BeautifulSoup``` sin necesidad de tener que buscar formas de navegar entre páginas e ir cogiendo lo que necesitamos de cada una de ellas.

Si investigamos la source de la página, podemos ver que los párrafos y el consecuente texto de El Quijote aparece en campos ```<p>```, si por tanto, le decimos ```soup.find_all("p")```, de acuerdo con [la documentación oficial](https://www.crummy.com/software/BeautifulSoup/bs4/doc/#find-all) filtramos los tags entre lo que nosotros queramos seleccionar, en este caso los parrafos.
Es aquí donde tenemos dos opciones con cada uno de los párrafos de ```soup``` (entendido como ```paragraph``` en el bucle) para tener el texto:

1. ```.get_text()```
> Éste método permite extraer todo el texto como un único string en Unicode. Podemos seleccionar una forma de separar el texto indicándolo en un primer parámetro e incluso podemos indicarle con un keyword que nos vaya eliminando el espacio en blanco y del final de cada párrafo con strip=True.
2. ```.stripped_strings```
> Éste método, por otro lado, automáticamente nos va quitando los espacios en blanco, haciéndonos más fácil la tarea de ir separando los párrafos e ir escribiendo sobre el archivo.

En este caso, el método ```.stripped_strings``` nos trae menos problemas, ya que ```.get_text()```, por algún motivo, nos multiplica *3 el texto total introducido y esto se ve reflejado en la cantidad de letras totales que contamos finalmente.

Avanzando un poco más, si bien es feo y tenga un gran gasto por cada ciclo, lo que que se me ocurrió para que dejarse de escribir en caso de llegar a un párrafo que no era parte del texto era hacer una comprobación del primer párrafo que va después del texto con el actual.

Con ésto, salimos de una forma (la verdad, muy fea, lo siento!) de nuestros loops y lo que hacemos, para no sobrecargar el tráfico de la página, es gracias al módulo ```time``` esperar durante un segundo para seguir el for exterior, pasando al siguiente índice.

Al final de este script, conseguimos tener un archivo ```Quijote.txt``` que contiene las dos partes de la misma obra. Es hora de pensar cómo procesarlo!


### De texto a.. troncho de texto!

Una vez que tenemos el txt, por qué no hacer otro txt donde tengamos las letras en minúsculas y sin acentos, haciéndonos más sencilla la cuenta? Vamos a ello!


Creamos el script siguiente:

```python
import unicodedata
special_characters = "!@#$%^&*()-+?_=,<>/.; "
words_of_Quijote = []
with open("Quijote.txt") as quijote, open("QuijoteEnCHUNK.txt","w") as quijote_csv:
#Guardamos todas las palabras en una lista de... muchisimos elementos.
    for line in quijote.readlines():
        if not (line == "\n"):
            temp_list = line.lower().strip(special_characters).replace(","," ").split()
            # quijote_csv.write("\n")
            for word in temp_list:
                if word in special_characters:
                    break
                if "ñ" in word:
                    temp_word = ""
                    for letter in word:
                        if letter == "ñ":
                            temp_word += (letter)
                        else:
                            temp_word += (unicodedata.normalize('NFD', letter).encode('ascii', 'ignore').decode("utf-8"))

                    quijote_csv.write(temp_word)
                else:
                    quijote_csv.write(unicodedata.normalize('NFD', word).encode('ascii', 'ignore').decode("utf-8"))
```

Lo que queremos hacer es lo siguiente:

1. Queremos tener una forma de convertir comprobar letra a letra si tenemos un carácter especial.
    1. En caso de ternerlo, queremos no escribirlo en el documento!
2. Queremos comprobar si nuestra palabra contiene la letra ñ, ya que es un caso especial y no queremos que el siguiente párrafo nos elimine la misma.
    2. Para ello, en el caso de las letras que contengan ñ queremos asegurarnos de que las demás letras si que no contienen ninguna tilde.
3. Con el módulo ```unicodedata``` codificamos a ASCII nuestra palabra para después volver a codificarla en UTF-8, eliminando así las tildes.

Con esto, tenemos una línea monstruosa de texto que ya podemos ir recorriendo (ya casi estamos ahí!)


### Contar las letras.

El último paso de este ejercicio es el de verificar que se cumple la frecuencia de las letras. Para ello, queremos comprobar si hemos cumplido con los datos que vienen en wikipedia (o al menos nos hemos acercado).

Para ello, este es el script que vamos a usar:

```python
total_words = 0

castellan = "abcedfghijklmnñopqrstuvwxyz"
count_dict = {}
with open("QuijoteEnCHUNK.txt") as quijote_csv:
    for troncho in quijote_csv.readlines():
        for letter in troncho:
            if letter in castellan:
                if letter not in count_dict.keys():
                    count_dict[letter] = 1
                total_words += 1
                count_dict[letter] += 1

print("\n Total palabras: {words}".format(words=total_words))
count_dict = dict(sorted(count_dict.items(), key=lambda item: item[1],reverse=True))
print(count_dict)

for key,value in count_dict.items():
    count_dict[key] = round(((value*100)/total_words),4)

print("\n-EN PORCENTAJE---\n")
count_dict = dict(sorted(count_dict.items(), key=lambda item: item[1],reverse=True))
print(count_dict)
```

En éste script, cogemos el texto escrito anteriormente, ```QuijoteENCHUNK.txt``` y nos ponemos a contar letra a letra para tener un número total ```total_words``` y un diccionario ```count_dict``` donde podemos asignar el número de ocurrencias para cada letra.

Una vez las tenemos, la imprimimos por pantalla en orden de aparición descendente, para así tenerlo en el mismo formato que en la wikipedia.

Si ejecutamos el script, obtenemos:

```
 Total palabras: 1590405
{'e': 222564, 'a': 194622, 'o': 157821, 's': 121876, 'n': 105273, 'r': 97660, 'i': 86918, 'l': 86258, 'd': 84625, 'u': 77060, 't': 59249, 'c': 57446, 'm': 43280, 'p': 34362, 'q': 31705, 'y': 24482, 'b': 23545, 'h': 19455, 'v': 17178, 'g': 16658, 'j': 10266, 'f': 7253, 'z': 6280, 'ñ': 4123, 'x': 469, 'w': 3}

-EN PORCENTAJE---

{'e': 13.9942, 'a': 12.2373, 'o': 9.9233, 's': 7.6632, 'n': 6.6193, 'r': 6.1406, 'i': 5.4651, 'l': 5.4236, 'd': 5.321, 'u': 4.8453, 't': 3.7254, 'c': 3.612, 'm': 2.7213, 'p': 2.1606, 'q': 1.9935, 'y': 1.5394, 'b': 1.4804, 'h': 1.2233, 'v': 1.0801, 'g': 1.0474, 'j': 0.6455, 'f': 0.456, 'z': 0.3949, 'ñ': 0.2592, 'x': 0.0295, 'w': 0.0002}
```

El cual, si no es el número total de letras que tiene El Quijote, nos acercamos con una diferencia de ~90K letras. Sin embargo, en el caso de las apariciones, vemos que efectivamente se cumple la aparición de las mismas con un poco más de resolución! :')



## Conclusiones

Este pequeño proyecto es una forma de buscar aplicar python a diversas ramas, como por ejemplo las de Data Sience o Teoría de la Información, donde en ésta segunda podríamos calcular la entropía media del castellano de cara a realizar coficaciones buscando códigos óptimos o formas de encriptar un texto basándonos en la [Longitud de Unicidad](https://es.wikipedia.org/wiki/Distancia_de_unicidad). Además, añadimos el hecho de usar una herramienta tan poderosa como es Beautiful Soup para poder obtener datos de páginas .html aunque estas en primera instancia no sean accesibles.

Si bien es cierto que no hemos conseguido el número exácto de palabras, ocurría que mediante el otro método ```.get_text()``` como hemos dicho antes, nos pasabamos al procudir el mismo 4M de letras, pero al final es cuestión de conseguir bien las etiquetas para obtener todo el texto que queramos.

Todo esto, claro está, se suma al poder decir a alguien viendo la ruleta de la suerte QUÉ LETRA DEBERÍA HABER COGIDO.
