**DECOMPILE**

Iniciamos la maquina:

<img width="567" height="350" alt="Imagen1" src="https://github.com/user-attachments/assets/92166fa3-279b-4cd3-af25-361bc2dc5615" />

**Escaneo de puertos y servicios**

Ejecutar un escaneo completo con Nmap para identificar los servicios expuestos:

<img width="567" height="653" alt="Imagen2" src="https://github.com/user-attachments/assets/0e47dcfa-b216-4f55-83f6-1e1baa91cd9a" />

<img width="567" height="525" alt="Imagen3" src="https://github.com/user-attachments/assets/31e57ced-f540-476d-939c-a37884c42404" />

**Con lo cual obtenemos:**

- Puerto **22/tcp** (SSH) abierto
- Puerto **80/tcp** (HTTP) abierto

**Exploración del servicio web**

Al acceder al servicio HTTP:

<img width="567" height="360" alt="Imagen4" src="https://github.com/user-attachments/assets/d03f06c0-2d03-4a13-aaa5-c0f1f13553cb" />

Descargamos el Apk:

<img width="567" height="441" alt="Imagen5" src="https://github.com/user-attachments/assets/9f71393a-8844-49f9-8c97-8619ce45e172" />

Dado que la pista mencionaba explícitamente un archivo .java, se utilizó **JADX**, herramienta especializada en recuperar código Java de binarios Android:

<img width="567" height="247" alt="Imagen6" src="https://github.com/user-attachments/assets/bced522f-9834-4d59-a435-8b14829a2ec6" />

Al inspeccionar la clase BuildConfig, encontramos la primera Flag y el API key:

<img width="567" height="216" alt="Imagen7" src="https://github.com/user-attachments/assets/336d4f43-a8e2-4c9d-a0c9-6a25d856959b" />

CTF_STAGE1 = "FLAG{stage1_buildconfig_leak_BKGVlytS}";

**Inspeccionando tenemos las vulnerabilidades identificadas:** CWE-798 (Use of Hard-coded Credentials) y CWE-312 (Cleartext Storage of Sensitive Information).

**Configuración de resolución local**

Añadimos una entrada en /etc/hosts para resolver el dominio interno:

<img width="567" height="95" alt="Imagen8" src="https://github.com/user-attachments/assets/571f1216-4231-4d0c-957d-dc1f623eda20" />

Intentamos acceder al endpoint base con la API Key descubierta:

<img width="886" height="247" alt="image" src="https://github.com/user-attachments/assets/03116d84-9282-41b2-b389-2e403dea5136" />

El servidor respondió indicando que no existían recursos en la raíz, lo que motivó un proceso de fuzzing.

<img width="886" height="562" alt="image" src="https://github.com/user-attachments/assets/3641a0ec-03ab-4c95-827b-d5f84d156709" />

**Segunda ronda de gobuster sobre /booking/ con la API Key**

Ahora enumeramos dentro de /booking/ **incluyendo el header de autenticación**

gobuster dir \\

\-u http://booking.internal/booking/ \\

\-w /usr/share/wordlists/dirb/common.txt \\

\-H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" \\

\-t 50
<img width="886" height="259" alt="image" src="https://github.com/user-attachments/assets/253792f2-0c26-46bc-9e01-f1af86ec2472" />
<img width="567" height="674" alt="image" src="https://github.com/user-attachments/assets/8d24ee02-260b-4fd1-814c-c33574eba34f" />

&nbsp;Se identificó el endpoint /properties.

**Acceso al endpoint /properties**

Consultamos manualmente el endpoint descubierto:
<img width="567" height="291" alt="image" src="https://github.com/user-attachments/assets/34adaa63-b55a-45e1-8a6c-78321968d845" />

**Respuesta del servidor (HTTP 200 OK):**

- Lista de propiedades
- **Información revelada (Information Disclosure):** "\_ctf_hint": "Auth OK. Try POST /sessions with tenant_account=INMO-MAD-PRIMARY"

**Vulnerabilidad identificada:** CWE-200 (Exposure of Sensitive Information).

**Creación de sesión y obtención de credenciales SSH**

Utilizando la información revelada, creamos una sesión válida mediante una petición POST:

<img width="567" height="261" alt="image" src="https://github.com/user-attachments/assets/6cc86e3a-c950-482d-9a52-3729a68e918a" />

La respuesta incluye la confirmación de la Fase 2 y revela en texto plano, las credenciales SSH:

- **Usuario:** deploy
- **Contraseña:** c0mpartir-p1s0s

Establecemos conexión SSH con las credenciales obtenidas:

<img width="567" height="321" alt="image" src="https://github.com/user-attachments/assets/99143667-af68-4040-be9b-bd6a48fffdfd" />

Vemos el directorio home

<img width="531" height="190" alt="image" src="https://github.com/user-attachments/assets/d00c9f28-4e08-496e-8934-a2fd0338b40c" />

Leemos la flag del usuario:

<img width="412" height="70" alt="image" src="https://github.com/user-attachments/assets/4e5b937d-f833-4517-9ceb-fe2f3efb4fea" />

stage3_ssh_user_land_08724a096e72

**Numeración de privilegios sudo**

Se verificaron los privilegios delegados del usuario con sudo -l:
<img width="886" height="272" alt="image" src="https://github.com/user-attachments/assets/63b0e4de-65a9-4dcf-95c9-39b572821e19" />


Tenemos dos binarios que puedes ejecutar como root sin contraseña. Cualquier binario que se ejecute como root y **no sanitice su entrada** es un candidato a escalada. Veamos el binario:

<img width="886" height="105" alt="image" src="https://github.com/user-attachments/assets/41773f2f-bdb3-4948-ad60-960e6a34a8e5" />

Esto significa que, al ejecutarlo, el proceso **adopta el UID del propietario** (root), **el binario ya es root por sí mismo**, sin necesidad de sudo.

Hacemos un análisis estático antes de ejecutar:

<img width="886" height="1208" alt="image" src="https://github.com/user-attachments/assets/5f155f1e-54b5-4b2a-b41f-d3732dc30f9a" />

Cuando pasamos "/bin/bash" como argumento, el binario construye:

text

logger ... ; /bin/bash

y lo ejecuta con bash -c **como root**. El logger es ruido; /bin/bash **después del** ; **se ejecuta como root**, shell root. Asi que lanzamos el binario con /bin/bash:

<img width="886" height="105" alt="image" src="https://github.com/user-attachments/assets/6075e31e-7e7f-40bb-8db5-2b55b5ab82a6" />

Confirmamos que ya somos root:

<img width="606" height="169" alt="image" src="https://github.com/user-attachments/assets/230222cd-f6aa-49cf-9ccb-2f7c1d2fa7ac" />

Con esto obtenemos la Flag 4:

<img width="809" height="163" alt="image" src="https://github.com/user-attachments/assets/8cc75b82-21be-4910-8baf-c4dbe3d6b2b4" />

FLAG{stage4_root_08724a096e72_1794010964}
