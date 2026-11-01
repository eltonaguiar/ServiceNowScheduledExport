# ServiceNowScheduledExport

<img width="1648" height="138" alt="image" src="https://github.com/user-attachments/assets/fbc8da03-d7b7-42cf-a818-0580b475fbed" />

<img width="267" height="35" alt="image" src="https://github.com/user-attachments/assets/9c2344c8-8e40-4cfe-ab1b-0d1a450646f0" />
Execute Now

Export Set components

Scheduled Data Export... 
<img width="1754" height="560" alt="image" src="https://github.com/user-attachments/assets/eea4b7a0-19c1-4c15-a0ef-1e31b8b4fa83" />
<img width="1642" height="526" alt="image" src="https://github.com/user-attachments/assets/ebeeb12c-425f-41ea-ac31-e9c9019b6367" />

There is a Delta Exports to only export the delta on or after Updated
<img width="1385" height="205" alt="image" src="https://github.com/user-attachments/assets/0a7d7eaa-2b95-49f7-bee3-62719ddcd24b" />
There is also a scripts section
<img width="1370" height="277" alt="image" src="https://github.com/user-attachments/assets/398acefe-4786-430b-a04f-0aa66cee881c" />

depends on 
Export Set

Which depends on Export Defintiion & Export Target
<img width="815" height="532" alt="image" src="https://github.com/user-attachments/assets/f909bf89-1f54-4cd8-ba32-8167ead1e6a4" />
<img width="808" height="481" alt="image" src="https://github.com/user-attachments/assets/428b7685-3a00-4695-b1f2-f5d339f28faa" />

Note also the MAX Rows: our events is currently at 8K, to make it future proof for now Ill put it at 100K max

for export set there is an append timestamp where I presume will append that to the file name
so torontoevents.csv would be 
torontoevents_2026_Nov_01_752AM.CSV somethign like that I guess..



