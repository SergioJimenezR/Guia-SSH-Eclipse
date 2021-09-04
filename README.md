# GESTIÓN DE CLAVES SSH EN GITHUB/ECLIPSE Y USO DE EGIT

Desde el pasado 13 de agosto de 2021, GitHub impuso una nueva política en cuanto a la autenticación. A partir de entonces, GitHub ya no admite hacer push/pull con solamente tu usuario y contraseña como se hacía antes, sino que ahora hace falta configurar claves SSH para ese doble factor de autenticación y acceder a los repositorios. Fuente oficial: https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/.

Esta guía se dividirá en tres grandes apartados:
1. ¿Qué son las claves SSH?. Contexto y entendimiento de las disposiciones de la clave privada y clave pública.
2. Generación y establecimiento de las claves, y otras configuraciones necesarias.
3. Caso práctico de cómo llevar a cabo el primer push desde Eclipse de un proyecto Maven situado en local, hacia un repositorio vacío de GitHub; y posterior clonación correcta.

Todo lo especificado se encontraría funcionando a fecha de sábado, 28 de agosto de 2021.

Solo a modo de aclaración, para lo relacionado con GitHub solamente **será necesario Eclipse y cualquier JRE o JDK instalado**. Además, para el contexto del punto 3 se podrá necesitar Maven y un JDK de la versión 7 de Java o en adelante. Las versiones exactas que se estarán utilizando en el caso y momento del tutorial y que funcionan correctamente, son las especificadas a continuación. En principio, al menos las posteriores versiones a éstas deberían de funcionar igualmente bien.
- **JDK**, en su `versión jdk1.8.0_301` para el caso del tutorial, o lo que es lo mismo, `jdk-8u301`. También funcionaría para la versión 7. No resulta necesario establecer las variables de entorno.
- **Eclipse IDE for Enterprise Java and Web Developers**, en su `versión 2021-06 (4.20.0)` y con `Build id: 20210612-2011`. Esta modalidad de Eclipse ya trae incluido Maven. Durante el tutorial se mantendrá configurado lo de por defecto, que es, la versión `16` en `Preferences > Java > Compiler`, y el JRE dispuesto por defecto en `Preferences > Java > Installed JREs` (aunque también se podrá añadir (a través del botón `Search`) y seleccionar el respectivo JDK necesariamente previamente instalado).
- No será necesario instalar Maven externamente si se utiliza ese Eclipse, puesto que ya lo trae. Aún así, igualmente para usar Maven se requeriría de `JDK 1.7 o posterior`. Tampoco sería necesario establecer las variables de entorno de Maven.

***

## 1. ¿Qué son las claves SSH?. Contexto y entendimiento de las disposiciones de las claves SSH.

### 1.1. ¿Qué es una clave privada y una clave pública?

Un par de claves asimétricas viene a dotar de una cierta seguridad, confidencialidad, integridad y, exclusividad del extremo de la clave privada, en el proceso de comunicación entre dos extremos dados.

En un contexto de criptografía, en general una de las claves cifra el mensaje, y solamente la otra lo puede descifrar. El quid de la cuestión está en que la clave privada se mantenga bajo confidencialidad y se utilice única y exclusivamente por el dueño de dicho par de claves, en este caso el cliente del servicio. Si dicha clave privada no se llega a usar por otras personas, sino que se mantiene en un mismo punto fijo, estaremos seguros de que ese extremo se trata exactamente de esa entidad (persona, máquina, etc.) en cuestión, y no de otra.

El par de claves no está asociado a algo en concreto, sino que se trata de una representación de una entidad (en general, tú), la cuál puede ser un servicio en ejecución o una máquina enteramente; y se puede usar para tantos sitios como se desee. La relación que existe entre la clave pública y la clave privada, es, que la pública circulará por aquellos sitios con los que te quieras comunicar de manera segura, utilizándose para cifrar un mensaje (salvo en los casos de firma), de manera que solamente tú con la clave privada (que no sale de tu máquina) puedes descifrarlo.

Por ejemplo, si la persona A utiliza la clave pública de la persona B para cifrar un mensaje, A se estará asegurando de que solamente la persona B sea la que lo vaya a poder leer, porque solo se puede descifrar con la clave privada y solamente la tiene B; proporcionando confidencialidad e integridad. Ésto es lo que se utiliza para el caso de GitHub y también de manera general. Salvo en procesos de firma, en los demás casos la pública es la que se suele utilizar para cifrar.

En otro símil de la clave pública, sería la que podrías divulgar en tu muro de Facebook para que tus amigos te cifren mensajes secretos que solamente tú con la privada puedes leer.

En cambio, en dichos procesos de firma, el sentido del cifrado es el contrario. Tanto monta, monta tanto. Suponiendo que la persona B cifra un mensaje con su clave privada, cuando la persona A lo descifre con la clave pública de B, A se estará asegurando de que la persona B ha sido exactamente quién lo ha enviado; proporcionando autenticación, integridad y no repudio en origen. Esto es lo que se utiliza por ejemplo en la firma digital. En este caso, se esta cifrando con la privada, y la pública descifra. Sin embargo, cabe destacar que el uso de la firma no garantiza la confidencialidad, puesto que cualquier entidad que dispusiese de la clave pública de B podría descifrar y obtener el mensaje que manda en texto plano.

Para el caso concreto de GitHub, éste usa tu clave pública como método de autenticación. Podemos decir de manera bastante abstraída que, el servidor [después de que se lo solicites, porque actúa bajo demanda] te manda un reto cifrado con tu pública y tú, para buscar demostrar que eres tú, teniendo tú que descifrarlo y responderle. Es decir, si me mandas una pregunta cifrada (con mi clave pública) que SOLAMENTE puedo descifrar yo (con mi clave privada), y soy capaz de contestar a ese reto (con la debida respuesta), es porque yo demuestro que soy quien digo ser; porque si fuese un impostor no podría descifrarla como para contestar correctamente.

### 1.2. Lugares de enfoque

En general la pública tiene como misión poder divulgarse sin peligro. Entregamos a GitHub la pública, y nosotros nos quedamos con la privada, porque si fuese al revés y GitHub se quedase con la privada, cualquiera con la pública podría hacerse pasar por nosotros en GitHub. Por eso, la idea es que solamente nosotros tengamos la clave privada.

Aún así, en realidad, en el mundo de la criptografía asimétrica, no existe ningún comportamiento reglado de cómo tienes que hacer uso o crear claves. En cada caso, un administrador de red decide qué cree que es mejor utilizar, cuántas crear, etc.

Es una práctica común que, si despliegas un servidor (supongamos que es HTTP para el ejemplo, aunque pudiera ser de cualquier tipo), se cree al menos un par de claves, cuya pública tenga asociado un certificado que indique en un campo `Subject` que ese certificado se está usando para un servidor HTTP. Pese a ésto, para el caso de Eclipse, la pública no lleva asociado un certificado, sino que se trata de un mero par de claves generado en Eclipse, y que a priori no iba ligado necesariamente a GitHub. En este caso de Eclipse, las claves se tratan de dos cadenas de 1024 bits (algo débiles, pero que no existe posibilidad de cambiarlo), que tienen la peculiaridad de que lo que cifra una, solo lo puede descifrar la otra, dando igual el sentido. Por definición, cualquiera de las dos podría servir como pública, pero seleccionamos una como pública directamente porque sería una pérdida de tiempo estar preguntando al creador cuando realmente da igual el sentido. Refiriéndome a bajo nivel, también existe un campo `palabra/frase de paso` que se introduce al crear el par y que viene siendo -para entendernos- algo así como una "contraseña" que nos pide cada vez que usamos la clave privada, para autorizar su uso. Por tanto, esto puede llegar a generar conflictos al momento de tener el mismo par de claves entre distintos usuarios SO o máquinas, porque el usuario SO en cuestión puede no saber qué "contraseña" utilizó el creador del par.

Para saber cuántas claves crear, en resumen y para simplificar, podremos decidir usar un par para cada usuario de SO. Veremos a continuación que es importante diferenciar entre usuario de SO y usuario de GitHub.

Al momento de generar las claves, veremos que las almacenaremos en un directorio en concreto, como puede ser el de ~/.ssh. A título de usuario de SO, podríamos decidir si compartir este par entre todos los usuarios de SO de la máquina, o no, porque los demás podrían tener, o no tener, acceso a ese directorio (para utilizar esa clave privada). Es por ésto que, para evitar posibles denegaciones de lectura por falta de permisos, se puede seguir la práctica de crear un par de claves por cada usuario de SO.

Aunque en otros contextos las claves puedan asociarse a título de servicio en lugar de a título de usuario de SO o de máquina, por simplificar podremos asignar un par de claves a cada máquina (o usuario SO), y no a cada usuario de GitHub.

Después, tenemos los usuarios de GitHub. Si se dispone de más de un usuario de GitHub, pero se está trabajando todo el rato desde el mismo usuario de SO, no es necesario crear más de un mismo par de claves, porque la privada va a la máquina y la pública se puede repartir tranquilamente entre los distintos usuarios de GitHub, de manera análoga a como si la estuviésemos repartiendo en distintas webs.

- Entonces, ¿se puede utilizar un mismo par para varios usuarios de SO? Por poder, en principio por supuesto que se debería de poder, pero si las claves están en un solo mismo directorio, habrá que asegurarse de que todos los usuarios tengan permiso de lectura.
- ¿Puedes usar un par para cada usuario de SO (lo que estábamos hablando)? También (de hecho, es lo más normal).
- Es más, como tercer caso, puedes tener tantos pares como quieras en un usuario de SO y decidir utilizar cada uno para una web/finalidad distinta si se es muy purista y quieres asegurarte de que si se compromete uno, no se comprometan los demás; aunque realmente, este último tercer caso de reparto puede ser algo exagerado y no suele darse demasiado a nivel normal.

Es decir, las claves no van asociadas necesariamente al usuario de un servicio ni tampoco a un usuario de SO. No tienen una etiqueta que explique que es de Pepe la cual pudiera limitar. Así que, se pueden distribuir, de manera oportuna, como se desee. Es decisión propia de cada uno cómo se quieren usar y cuántas crear.

¿Se pueden tener dos claves en el mismo usuario de SO? Como tal, no habría ningún problema. Pero si lo que te preocupa es poder diferenciar claramente entre distintos usuarios de GitHub del mismo usuario de SO, te diré que se podrá seleccionar fácilmente en Eclipse el usuario de GitHub que commitea (e igualmente tendrías que seleccionarlo aunque hubiesen más claves). Es por ello que diferenciamos a título de usuario de SO, y no de usuario de GitHub.

Evidentemente, si una persona con un usuario de GitHub utiliza varias máquinas o usuarios de SO, podrá disponer de un par para cada máquina o usuario de SO. No hay ningún problema en establecer varias públicas en GitHub.

En general, ¿cuál es mi consejo para las personas, las cuales, este mundo no sea su preocupación diaria? Que tengan un par por cada usuario de SO. Es decir, que si existen varios usuarios SO en una misma máquina, se cree un par para cada usuario SO; y que si dentro de éste, lo utilizan varios usuarios de GitHub, se aplique la misma pública para todos.

Los campos `user.name` y `user.email` que veremos a continuación se usan para decir quién ha hecho el commit (que luego, en GitHub figura exactamente este nombre). Con respecto a lo de seleccionar el usuario de GitHub que commitea, en caso de existir varios, aunque en la configuración de Eclipse (en `Preferences`) establezcamos el usuario habitual concreto (y ese lo dejemos ahí fijo), podremos seleccionar justo en el momento de hacer el commit qué usuario nos interesa que commitee. Si nos fijamos, en la esquina inferior derecha de la pestaña `Git Staging` disponemos de dos recuadros de texto, autorellenados con los de la configuración establecida, pero modificables. Es decir, exactamente ahí en esos dos recuadros, podremos cambiar el usuario que commitea por el que queramos, en cualquier momento, sin tener que cambiar el de la configuración (lo cual podría resultar más tedioso).

¡Importante! Si utilizamos distintos IDE con respecto a GitHub y tuviésemos que establecer el par en varios sitios, como estamos decidiendo identificarnos según máquina (o usuario de SO) y no según servicio, nos servirá el mismo par para varios procesos (servicios). Es decir, podrás configurar la misma clave privada en Eclipse y en Visual Studio Code si fuese necesario.

***

## 2. Generación y establecimiento de las claves, y otras configuraciones necesarias.

### 2.1. ¿Cómo establecer el usuario de GitHub en Eclipse?

En primer lugar será necesario asegurarnos de que tenemos establecido un usuario de GitHub en Eclipse. Para ello, procederemos con sucesivos clicks en `Window > Preferences > Version Control (Team) > Git > Configuration > User Settings`. Si disponemos de un mapeo de `user` con sus dos respectivos pares clave-valor `email` y `name`, ya está hecho. En caso contrario, lo deberemos considerar pulsando en `Add Entry` y estableciendo estos dos pares clave-valor:

**Par n.º 1:** Key: `user.email`. Value: Tu correo electrónico de GitHub. Ejemplo: `sergio@gmail.com`.

**Par n.º 2:** Key: `user.name`. Value: Tu nombre de usuario de GitHub. Ejemplo: `SergioJimenezR`.

No resulta necesario considerar los pares clave-valor de la key `filter` que se podrían encontrar en otras configuraciones.

Finalmente, procedemos a pulsar `Apply and Close`. Probablemente si ya tuviéramos previamente un archivo `.gitconfig` generado, apareciesen ya esos pares clave-valor. En caso contrario, ocurriría que aunque Eclipse nos indicase una ruta del `.gitconfig`, no existiese realmente dicho archivo allí antes de llevar a cabo este proceso. En cualquier caso, tras ésto, se habrá generado automáticamente en dicha ruta y no tenemos que preocuparnos.

### 2.2. ¿Cómo generar las nuevas claves RSA/SSH con ayuda de Eclipse?
Nótese que la localización del Workspace resulta totalmente indiferente a la generación y configuración de claves, y no se perderá aunque cambiemos de Workspace, ya que esta configuración se mantiene guardada en los archivos de Eclipse y no en la carpeta `.metadata` del Workpace.

En Eclipse, procedemos con sucesivos clicks en `Window > Preferences > General (desplegamos) > Network Connections (desplegamos) > SSH2`. A continuación, continuamos con sucesivos clicks en la pestaña `Key Management > Generate RSA Key...`. Podemos observar cómo se acaba de generar el par de claves. Tras haberlas generado, pulsaremos en `Save Private Key` para guardarlas. Pulsaremos `OK` en ambos cuadros de diálogo. Las podremos guardar en la carpeta oculta `C:/Users/Usuario/.ssh` que se nos acaba de generar, o en general donde queramos. Se tratan de dos archivos de texto plano llamados `id_rsa` (clave privada) e `id_rsa.pub` (clave pública). El archivo con extensión `.pub` es la clave pública, mientras que la otra es la clave privada.

### 2.3. ¿Cómo y dónde establecer la clave privada y pública?
Estableceremos por un lado la **clave privada `id_rsa` asecas en el lado de Eclipse**. Para ello, en el mismo apartado `SSH2` de la ventana `Preferences` en la que estábamos antes, pero esta vez en la pestaña `General` en lugar de en `Key Management`, pulsaremos en `Add Private Key` y seleccionaremos el archivo de la clave privada (`id_rsa` asecas, sin `.pub`) (¡ojo, diferenciad claramente la privada de la pública, a través de la extensión `.pub`!), localizando el archivo allá donde lo hayamos decidido guardar. Es posible que ya apareciese como añadido, asegurémonos pues. Tras ello procedemos con `Apply and Close`.

Por otro lado, estableceremos la **clave pública `id_rsa.pub` en GitHub**. Para ello, evidentemente iniciaremos sesión, y a través del menú que se despliega clickando en nuestro avatar de la esquina superior derecha, accederemos al apartado de `Settings`. Después, en el apartado `SSH and GPG keys` pulsaremos en `New SSH key`, proporcionándole manualmente en el apartado `Title` un título lo suficientemente aclarativo e identificativo de la máquina donde vamos a utilizar Eclipse. Después, copiaremos y pegaremos exactamente **todo el contenido** del archivo `id_rsa.pub` (abierto con un editor de texto plano) en el recuadro de `Key`. Finalmente pulsaremos en `Add SSH key`.

Para proceder utilizando este sistema en lugar del clásico, la diferencia principal en cuanto al uso que le damos, consiste en, utilizar la `URI SSH` que nos proporciona GitHub, en lugar de la `URI HTTPS` que se venía utilizando con anterioridad, cuando, al enlazar con respecto al repositorio, Eclipse nos pedía la URI; y además, en esa misma ventana de la URI deberemos mantener el usuario `git` (sin contraseña) que se nos autorellena al colocar ésta. (Directamente continuaremos pulsando `Next, Next... Finish` de manera general, o a nuestro gusto).

Nótese que si ya teníamos desplegados repositorios en local en la pestaña `Git Repositories` de Eclipse, deberemos de eliminar los `Remote` de los mismos, ya que figurarían según el sistema `HTTPS` y no proporcionaría acceso, para que así cambien al `SSH` automáticamente.

***

## 3. Caso práctico de cómo llevar a cabo el primer push desde Eclipse de un proyecto Maven situado en local, hacia un repositorio vacío de GitHub; y posterior clonación correcta.

### 3.1. Primer push
Deberemos disponer del proyecto en cuestión situado en local en nuestro Workspace (el cual puede ser de cualquier tipo: Java Project, Maven Project (para Maven se recomienda de manera general el uso del arquetipo `maven-archetype-quickstart`), etc.), y del repositorio vacío en GitHub.

Click derecho sobre el proyecto en `Eclipse > Team > Share Project > Create`. Donde la ruta, dejaremos indicada una ruta que sea donde se vaya a almacenar todo lo relacionado con respecto al repositorio local. Nótese que si seleccionamos en `Browse` un repositorio local existente, lo estaremos reutilizando. Personalmente recomiendo `C:\Users\Usuario\git\nombreDelRepositorio` para la ruta. El repositorio local no tiene por qué llamarse igual que el proyecto, aunque puede hacerlo; pero sí que se recomienda que se llame igual que el repositorio de GitHub. Tras ello, `Finish`, y de nuevo `Finish`.

Tras ésto, se nos habrá creado el repositorio en local en dicha ruta, y deberemos commitearlo con el de GitHub haciendo un simple `Commit and Push` en Eclipse hacia su `URI SSH`.

Para ello, primeramente se recomienda disponer de las pestañas `Git Repositories` y `Git Staging`. Si no disponemos de ellas, procederemos a activarlas yendo a `Window > Show View > Other > Git > (seleccionando ambas con Ctrl) Git Repositories y Git Staging > Open`.

Nos dispondremos a realizar un `Commit and Push` normal y corriente, con la diferencia de que, llegado un momento dado, nos pedirá la `URI SSH`. Existen varias alternativas para llevar a cabo ésto. Una de ellas es la siguiente: Desde la pestaña de `Git Staging`, seleccionaremos los respectivos archivos para pushear, indicaremos la información del commit, los campos de `Author` y `Committer` deberían aparecer automáticamente (tal y como establecimos los pares clave-valor `user.email` y `user.name` al principio), y click en `Commit and Push`. Justo aquí, se nos debería de abrir una ventana. En ella, situaremos la `URI SSH` extraída del repositorio de GitHub, y a continuación DIRECTAMENTE click en `Preview`. En cada uno de los dos cuadros de diálogo pequeños que nos aparezcan, marcaremos la respectiva casilla y pulsaremos `OK`. De ellos, el primero de los dos es para que se quede guardada la configuración de la clave SSH, y el segundo es acerca de un archivo que guarda con quién te comunicas vía SSH. Finalmente continuaremos con el Push de manera normal, o a nuestro gusto. Nótese que podrás seleccionar `Rebase interactively` en el desplegable de `When pulling:` en lugar del clásico `Merge` si así lo prefieres para cuando vayan a haber conflictos en los pull que hagas de este repositorio en local. Click en `Preview` y `Push` para finalizar el push.

Una vez realizado este primer Push, los archivos deberían encontrarse subidos al repositorio de GitHub.

### 3.2. Clone
Los demás integrantes del equipo (que evidentemente deberán ser `Collaborators` en el repositorio si éste es privado) podrán realizar un clonado del repositorio con ayuda de la `URI SSH` del mismo una vez también hayan establecido lo necesario con respecto a la autenticación SSH explicado en el anterior apartado.

Para llevar a cabo el clonado del repositorio se recomienda importarlo. Para ello, desde Eclipse iremos a `File > Import > Git > Projects from Git (el que NO es smart) > Clone URI`. En este instante, especificaremos la `URI SSH` en el campo de `URI`, se nos autorellenará el usuario con `git` (y sin contraseña) y pulsaremos en `Next`. Continuaremos con el clone a nuestro gusto, pudiendo seleccionar la respectiva rama Branch, el directorio donde queramos alojar el repositorio, y sucesivos `Next`. Generalmente `Next, Next... Finish` salvo particularidades concretas. Tras el `Finish`, nos debería de aparecer este nuevo proyecto en `Project Explorer` y el respectivo repositorio en `Git Repositories`. Además, si se trata de Maven, deberá señalarse cargando (`Building`) abajo a la derecha, y nosotros esperar hasta el fin de la carga.

Si no funcionase, también existen otras formas para importar un proyecto de un repositorio remoto, como por ejemplo, clonar el repositorio en la pestaña de `Git Repositories`, y después de haberlo clonado, darle click derecho e `Import Projects`.

Para llevar a cabo posteriores Push, los podremos hacer desde la pestaña `Git Staging`. Y los Pull, a través de: `Click derecho en el respectivo proyecto > Team > Pull`. ¡Cuidado con los conflictos de Merge!.

Si se eliminasen por completo el proyecto y el repositorio local, se recomienda reiniciar Eclipse antes de volver a clonarlo/importarlo.

***

[Sergio Jiménez Roncero](https://www.linkedin.com/in/sergiojimenezr/)

Visitas: ![](https://komarev.com/ghpvc/?username=SergioJimenezR&color=blue)
