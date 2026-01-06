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



