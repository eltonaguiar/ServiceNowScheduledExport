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
EXPORT DEFINITION
<img width="1520" height="349" alt="image" src="https://github.com/user-attachments/assets/ad01fa24-7d75-46f4-a5f0-54c29f185288" />

<img width="815" height="532" alt="image" src="https://github.com/user-attachments/assets/f909bf89-1f54-4cd8-ba32-8167ead1e6a4" />
<img width="808" height="481" alt="image" src="https://github.com/user-attachments/assets/428b7685-3a00-4695-b1f2-f5d339f28faa" />
#EXPORT TARGET 
https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_export_target.do%3Fsys_id%3Da57114edc3d09750310abf0d050131bf%26sysparm_record_target%3Dsys_export_target%26sysparm_record_row%3D1%26sysparm_record_rows%3D1%26sysparm_record_list%3DORDERBYname

Note the mid server specified then file path
e.g. file path of  toronto/events  with the "export set" configured with "append time stamp" and maximum rows 100,000 and CSV format, coolumn headers=use field name, column values=use display value

the exports go here
"C:\ServiceNow\MID\agent\export\toronto\events\torontoevents_20261101050335.csv"
"C:\ServiceNow\MID\agent\export\toronto\events\torontoevents_20261101050406.csv"

<img width="630" height="172" alt="image" src="https://github.com/user-attachments/assets/4fa89ff8-977c-48c6-a24a-a2a9fff6b532" />

"u_location","u_event_id","u_last_updated","u_event_date","u_title","sys_created_on","sys_created_by","sys_tags","sys_updated_on","sys_updated_by","sys_mod_count","sys_id"

**Blank Export**
[torontoevents_20261101050335.csv](https://github.com/user-attachments/files/34238757/torontoevents_20261101050335.csv)
"u_location","u_event_id","u_last_updated","u_event_date","u_title","sys_created_on","sys_created_by","sys_tags","sys_updated_on","sys_updated_by","sys_mod_count","sys_id"

(see the .csv attachment with records for the export that worked)



Note also the MAX Rows: our events is currently at 8K, to make it future proof for now Ill put it at 100K max

for export set there is an append timestamp where I presume will append that to the file name
so torontoevents.csv would be 
torontoevents_2026_Nov_01_752AM.CSV somethign like that I guess..




