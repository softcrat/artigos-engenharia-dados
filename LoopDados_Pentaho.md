<img width="1096" height="272" alt="image" src="https://github.com/user-attachments/assets/bab1cdaa-2e22-4516-9132-fa13c7075a4f" />
Na ranformação ListarAnos
<img width="538" height="135" alt="image" src="https://github.com/user-attachments/assets/0ca3dce6-a3f4-4186-83bc-2fdc1e688097" />
Sql Input Table usando o Postgresql
select 
*
from(
SELECT CAST(EXTRACT(YEAR FROM  CURRENT_DATE + INTERVAL '-2 Year') AS varchar(5)) as ano
UNION
SELECT CAST(EXTRACT(YEAR FROM CURRENT_DATE + INTERVAL '-1 Year') AS varchar(5)) as ano
UNION
SELECT CAST(EXTRACT(YEAR FROM CURRENT_DATE + INTERVAL '0 Year') AS varchar(5)) as ano
) r 
order by ano 
O step Copy rows to result vai copiar o resultado para a memória
Subjob Que processa os dados
<img width="866" height="289" alt="image" src="https://github.com/user-attachments/assets/2a0ddf26-2631-4567-8949-e4cc2fb12fc7" />

A tranformação processamento dos dados ficou assim:
<img width="1257" height="173" alt="image" src="https://github.com/user-attachments/assets/6e1898c5-4238-4562-a677-652779c5d1d0" />


<img width="945" height="221" alt="image" src="https://github.com/user-attachments/assets/22a290e7-b1a0-4148-934c-ba968a3b3641" />
<img width="924" height="263" alt="image" src="https://github.com/user-attachments/assets/a7d71438-f2ff-401b-be58-9bc63483285a" />
<img width="779" height="162" alt="image" src="https://github.com/user-attachments/assets/8be32ffe-7a46-4c20-a002-9fc4f30ad3ad" />


