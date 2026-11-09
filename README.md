# Proiect facultate 2026

## Talking Avatar 3D · Manual de predare, utilizare și susținere

**Versiunea documentată** — aplicația portabilă Windows x64 din arhiva `VOX-Portable-Windows-x64.zip`  
**Data documentației** — 9 noiembrie 2026

### 1. Rezumat și obiectiv

Aplicația pune în legătură grafica 3D, sinteza vocală în limba română și generarea de text cu un model lingvistic local. Utilizatorul poate scrie un text pe care avatarul îl rostește direct sau poate pune o întrebare. În al doilea caz, modelul generează răspunsul, Piper produce sunetul, iar Unity animează fața în timp ce sunetul este redat. Un workflow n8n sau Flowise poate declanșa aceleași operații prin HTTP.

Problema tehnică principală este coordonarea a trei procese cu ritmuri diferite. Modelul lingvistic livrează fragmente de text, sinteza vocală lucrează pe fraze, iar grafica trebuie să se actualizeze continuu. Proiectul separă aceste responsabilități și folosește **poziția din clipul audio** ca referință temporală pentru mișcarea gurii.

Produsul final se execută pe Windows 11 x64 dintr-un folder extras. Pachetul conține componentele necesare pentru demonstrația locală offline, inclusiv modelele. Proiectul sursă Unity este un livrabil separat de arhiva de execuție.

### 2. Arhitectura la rulare

```text
                 +----------------------------+
                 | Unity · VOX.exe · fereastră|
                 | UI, coadă, audio, avatar 3D|
                 +-------------+--------------+
                               |
            +------------------+------------------+
            |                                     |
     întrebare / flux text                  frază / WAV + viseme
            |                                     |
     Ollama local                            Piper local
     lfm3-omni / gemma4                      Lili / Raluca / Sanda

     Chatbox sau formular → n8n / Flowise → API local → coadă → Unity
```

| Componentă | Rol | Date la intrare și ieșire |
|---|---|---|
| Unity 6.6, C#, Windows x64 | Interfață, afișarea scenei, animația facială și redarea audio | Text, clipuri WAV și indicații temporale → cadre 3D și sunet |
| Ollama | Rulează modelul lingvistic | Prompt JSON → fragmente de răspuns prin HTTP |
| Piper, ONNX Runtime | Sintetizează vorbirea în română | Text și voce → PCM în container WAV și o cronologie de viseme, dacă este disponibilă |
| Python 3.13 inclus | Rulează lansatorul, serviciul TTS și puntea API | Configurație locală, procese și cereri HTTP |
| API VOX | Primește comenzi externe pentru avatar | JSON → ID de cerere, stare și text final |

Modul `speak` ocolește etapa Ollama. În modul `ask`, textul final depinde de modelul selectat. Interfața mai permite configurarea unui server **LM Studio** compatibil, pentru care aplicația interpretează fluxul SSE. Serviciul LM Studio se furnizează separat. Configurația offline livrată folosește Ollama. [Contractul Ollama Chat API](https://docs.ollama.com/api/chat)

### 3. Ce conține livrarea

```text
VOX-Portable/
├─ Start-VOX.cmd
├─ launch.py
├─ api_bridge.py
├─ Avatar/
│  ├─ VOX.exe
│  ├─ VOX_Data/
│  ├─ UnityPlayer.dll
│  └─ MonoBleedingEdge/ și biblioteci native
├─ Piper/
│  ├─ server.py
│  ├─ Python/
│  └─ models/
│     ├─ ro_RO-lili-high.onnx și .onnx.json
│     ├─ ro_RO-raluca-high.onnx și .onnx.json
│     └─ ro_RO-sanda-high.onnx și .onnx.json
├─ Ollama/
│  ├─ ollama.exe
│  ├─ lib/
│  └─ models/  (manifeste și bloburi pentru cele două modele)
├─ Licenses/
├─ CITEȘTE-MĂ.md
└─ API-n8n-Flowise.md
```

La rulare apar `Logs/` și `API-token.txt`. Tokenul este generat local la prima pornire a API-ului. Modelele Ollama incluse sunt **`lfm3-omni:latest`** și **`gemma4:latest`**. Eticheta `latest` desemnează copia împachetată, identificată de manifestul inclus. Lansatorul verifică prezența modelelor locale, fără să le înlocuiască automat cu versiuni noi.

Arhiva are aproximativ **13 GB**. Pentru arhivă și conținutul extras pe același disc, se rezervă aproximativ **27 GB**. Se extrage integral pe un volum NTFS sau exFAT, într-un director cu drept de scriere. Structura relativă a subfolderelor trebuie păstrată. Performanța inferenței depinde de RAM, procesor, GPU și drivere. Cele două modele se încarcă la cerere, pe rând.

### 4. Pornire și parametrii lansatorului

Din PowerShell, în directorul extras `VOX-Portable`:

```powershell
.\Start-VOX.cmd
.\Start-VOX.cmd --help
.\Start-VOX.cmd --self-test
.\Start-VOX.cmd --cpu
.\Start-VOX.cmd --cpu --self-test
```

| Argument | Efect |
|---|---|
| Fără argumente | Pornește Ollama, Piper și API-ul, apoi deschide avatarul. |
| `--help` sau `-h` | Afișează opțiunile declarate de parser. |
| `--self-test` | Pornește Piper și Ollama, sintetizează un exemplu cu fiecare voce și cere un răspuns fiecărui model. Afișează rezultatele `PASS` dacă probele reușesc. Testul rulează fără fereastra avatarului și fără puntea API. |
| `--cpu` | Configurează inferența Ollama pentru CPU. Afectează serviciul de modele, iar Unity continuă să folosească grafica Windows. Poate fi combinat cu `--self-test`. |

Forma corectă este **`--self-test`**, cu două cratime. Opțiunile sunt declarate în **`VOX-Portable/launch.py`**, în funcția `main()`, prin `parser.add_argument(...)`. `Start-VOX.cmd` găsește Python și scriptul relativ la folderul propriu, prin `%~dp0`, și transmite argumentele cu `%*`. Fișierul CMD folosește linii Windows CRLF. Parametrii `--model` și `--port` aparțin separat lui `Piper/server.py`; lansatorul îi gestionează pe cei necesari pentru pornirea portabilă.

**Succesiunea de pornire.** Lansatorul rezervă două porturi locale disponibile, pornește Ollama și Piper, apoi verifică modelele prin `/api/tags` și vocile prin `/health`. În modul normal pornește puntea API la `127.0.0.1:8765` și transmite playerului adresele serviciilor. Fereastra este deschisă după ce verificările reușesc. La închiderea avatarului, lansatorul oprește serviciile pe care le-a pornit.

| Setare | Loc și efect |
|---|---|
| `OLLAMA_MODELS` | `launch.py`. Trimite Ollama către depozitul din pachet. |
| `OLLAMA_HOST`, `VOX_OLLAMA_URL`, `VOX_PIPER_URL` | `launch.py`. Adrese locale, stabilite pentru sesiunea curentă. |
| `OLLAMA_NO_CLOUD=1` | `launch.py`. Modul de cloud Ollama este dezactivat. |
| `OLLAMA_CONTEXT_LENGTH=4096` | `launch.py`. Configurează contextul implicit al serverului local. |
| `OLLAMA_MAX_LOADED_MODELS=1` și `OLLAMA_NUM_PARALLEL=1` | `launch.py`. Limitează modelele simultan încărcate și paralelismul pentru a controla consumul de memorie. |
| `PORT = 8765` | `api_bridge.py`. Port fix pentru automatizări. Lansatorul raportează dacă este ocupat. |
| `-force-d3d11` | Argument pentru playerul Unity, transmis de lansator pentru randarea Windows. |

Din interfață se aleg avatarul, vocea și modelul. **Rostește textul** testează TTS fără generare, iar **Întreabă modelul** testează întregul flux. **Oprește** întrerupe operația. **F11** comută ecranul complet, iar **Esc** revine în fereastră. Maximize și Restore sunt comenzi native Windows. Când fereastra este îngustă, comenzile se așază sub portret.

### 5. Cum a fost construit proiectul

1. **Scena 3D.** Cele trei modele furnizate au fost importate ca FBX. Materialele au primit texturi și măști de opacitate acolo unde a fost necesar. Fiecare model a fost poziționat în raport cu camera de portret. Iluminarea folosește lumină principală, de umplere și de contur.
2. **Interfața și scena de test.** Scena `Assets/Portrait/VOX Portrait.unity` conține camera, avatarele și controalele. Scena permite testarea manuală a visemelor, selectarea avatarului și rotirea portretului. Adresele serviciilor sunt separate de configurația scenei.
3. **Fluxul de text.** C# trimite promptul și instrucțiunea de dialog la model. Ollama furnizează obiecte JSON pe linii, iar aplicația acumulează fragmentele. Textul este segmentat la limite de frază sau la spații după acumularea unui fragment mai lung. Fragmentele complete intră în coada de rostire înainte de sfârșitul întregului răspuns. Sinteza și redarea frazelor sunt ordonate.
4. **Fluxul audio.** Pentru fiecare fragment, Unity trimite către Piper `text` și identificatorul vocii. Piper încarcă modelul de voce la nevoie și răspunde cu WAV. Unity creează un `AudioClip` și îl redă prin `AudioSource`. Motorul Piper 1.7.0 rulează separat de player. [Sursa Piper](https://github.com/OHF-Voice/piper1-gpl)
5. **Controlul facial.** Piper poate extrage duratele fonemelor din graful ONNX al vocilor din pachet. Timpii sunt grupați în viseme și trimiși odată cu sunetul. Unity interpolează ponderile sau pozițiile articulațiilor pentru a reduce salturile între cadre.
6. **API și pachetul portabil.** Puntea HTTP trimite joburi către player, care folosește aceeași cale de redare ca butoanele din interfață. Playerul a fost compilat din Unity **6000.6.8f1** pentru `StandaloneWindows64`. Arhiva reunește rezultatul buildului, bibliotecile, interpretorul, serviciile și modelele necesare demonstrării locale.

**Rațiunea segmentării.** Citirea fiecărui token separat ar strica pronunția și ritmul. Așteptarea răspunsului complet ar amâna începutul vorbirii. O frază oferă context suficient vocii și permite ca primul clip să fie redat în timp ce modelul continuă să genereze.

### 6. Sincronizarea buzelor și randarea

Un **fonem** reprezintă o unitate sonoră. O **visemă** reprezintă o configurație vizibilă a gurii. De exemplu, /m/, /b/ și /p/ folosesc închiderea buzelor și pot împărți aceeași visemă.

| Grup | Mișcare urmărită |
|---|---|
| A și vocale centrale | Deschidere mai mare |
| E, I | Deschidere mai mică, întindere laterală |
| B, M, P | Închiderea buzelor |
| F, V | Poziție labiodentală |
| O, U | Rotunjire |
| Alte consoane | Poziție de articulare simplificată |
| Pauze | Revenire către poziția neutră |

Pentru modelele Piper din pachet, ieșirea de durată a fost expusă prin tensorul existent `/Ceil_output_0`. Ponderile vocilor au rămas aceleași. Serviciul transmite indicații de forma `visemă:număr_de_eșantioane` în headerul `X-Vox-Visemes`. Unity verifică dacă suma duratelor corespunde lungimii clipului. La fiecare cadru, selectează intervalul care conține `AudioSource.timeSamples`. Dacă marcajele lipsesc sau nu trec verificarea, aplicația distribuie visemele estimate din text pe durata clipului.

- **Adriana** folosește formele faciale ale modelului, inclusiv țintele de clipit. O formă de deformare, numită *blendshape*, deplasează vârfurile mesh-ului în funcție de o pondere. Formele faciale ale Adrianei includ deja mișcarea maxilarului. Deformarea separată a maxilarului se aplică pe mesh-urile fără viseme native, precum cel al dinților.
- **Alexia**, modelul Toon3D, folosește articulația maxilarului. Articularea este mai simplă decât la Adriana.
- **Alexis**, modelul Witch, are articulații pentru maxilar, buzele de sus și de jos și colțurile gurii. Acestea sunt coordonate din aceeași cronologie.

Interpolarea folosește un factor dependent de durata cadrului, de forma `1 − exp(−k × Δt)`. O rată mai mare reduce întârzierea cu care forma gurii urmărește fonemele scurte. Interpolarea păstrează tranziția lină între poziții.

Camera portretului randează într-o **RenderTexture**, adică o textură produsă de cameră. Aceasta este afișată într-o zonă distinctă de panoul de comenzi. Dimensiunea zonei se recalculează când se modifică fereastra. Această cale de afișare a rezolvat cazul în care Adriana apărea în randarea camerei, dar lipsea din Game View.

### 7. Integrare prin API cu n8n sau Flowise

Un chatbox poate apela webhook-ul workflow-ului. Workflow-ul alege apoi dacă trimite textul final pentru rostire sau trimite întrebarea către modelul VOX. Apelul se face din backend, cu tokenul păstrat în credențialele workflow-ului.

| Operație | Endpoint | Răspuns |
|---|---|---|
| Disponibilitate | `GET /api/v1/health` | Starea serviciului |
| Trimitere | `POST /api/v1/jobs` | HTTP 202, `id`, `mode` și stare inițială |
| Citire rezultat | `GET /api/v1/jobs/<id>` | `queued`, `running`, `done` sau `error` |

Fiecare apel folosește `Authorization: Bearer <token>`. Pentru POST se trimite `Content-Type: application/json`. Tokenul se citește din `API-token.txt`. Exemplu din PowerShell, cu aplicația pornită:

```powershell
$base = 'http://127.0.0.1:8765/api/v1'
$token = (Get-Content .\API-token.txt -Raw).Trim()
$headers = @{ Authorization = "Bearer $token" }
$payload = @{ mode = 'speak'; text = 'Bună! Formularul a fost primit.' } |
    ConvertTo-Json -Compress

$job = Invoke-RestMethod "$base/jobs" -Method Post -Headers $headers `
    -ContentType 'application/json; charset=utf-8' -Body $payload
Invoke-RestMethod "$base/jobs/$($job.id)" -Headers $headers
```

Interogarea se repetă după o pauză până la o stare finală. `speak` citește textul primit, iar `ask` îl tratează ca prompt. La `done`, `answer` conține textul rostit sau răspunsul generat. Coada are maximum 20 de cereri în așteptare. HTTP 401 semnalează autentificare absentă sau invalidă, iar 429 indică o coadă plină. Datele cererilor sunt păstrate în memoria procesului pe durata sesiunii. Cererile blocate pot expira.

**Acces din rețea.** Adresa `127.0.0.1` desemnează calculatorul pe care rulează apelantul. Dacă n8n sau Flowise rulează în cloud ori în alt mediu de rețea, trebuie configurată separat o punte sau un proxy privat către calculatorul VOX. Browserul chatboxului trimite date către backend, iar backendul păstrează tokenul. Pachetul pornește cu API-ul legat de loopback și respinge cererile directe cu origine web. [Nodul HTTP Request din n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) · [Nodul HTTP din Flowise Agentflow V2](https://docs.flowiseai.com/using-flowise/agentflowv2)

### 8. Harta surselor pentru susținere

| Fișier | Unde se vede decizia tehnică |
|---|---|
| `Assets/Editor/VoxScene.cs` | Instanțierea celor trei modele, materialele, iluminarea și camera |
| `Assets/VoxPortrait.cs` | Selectorii, comenzile, setările, pornirea operațiilor și ecranul complet |
| `Assets/VoxWindowLayout.cs` și `VoxGameOutput.cs` | Adaptarea la rezoluție, randarea și compunerea portretului |
| `Assets/VoxConversation.cs` | Streaming, segmentarea frazelor, solicitarea WAV, redarea și alegerea visemei |
| `Assets/VoxAvatar.cs` și `VoxLipRig.cs` | Aplicarea blendshape-urilor și deplasarea maxilarului sau a buzelor |
| `Assets/VoxApiWorker.cs` | Preluarea cererilor externe și trimiterea stării finale |
| `TTS/server.py` în sursă, `Piper/server.py` în pachet | Selecția vocii, sinteza, formatul WAV și metadatele de sincronizare |
| `Start-VOX.cmd`, `launch.py`, `api_bridge.py` | Pornirea, parametrii, coordonarea proceselor și contractul HTTP |

Pentru modificarea scenei se deschide proiectul sursă în editorul Unity 6000.6.8f1, cu suport **Windows Build Support**, și se încarcă `Assets/Portrait/VOX Portrait.unity`. Pentru redistribuire se compilează Windows x64, se înlocuiește împreună playerul și directorul `VOX_Data` aferent în structura de livrare, apoi se verifică din nou lansatorul dintr-o copie extrasă. Editorul se folosește la dezvoltare și recompilare. Pentru demonstrație se transmite pachetul executabil.

### 9. Verificare și depanare

| Simptom | Verificare |
|---|---|
| Lansatorul raportează fișiere lipsă | Confirmă extragerea completă și prezența `Piper/Python/python.exe`, `launch.py` și `Avatar/VOX.exe`. |
| Fereastra întârzie la pornire | Consultă `Logs/ollama.log` și `Logs/piper.log`. Lansatorul așteaptă răspunsul serviciilor. |
| Prima întrebare durează mai mult | Modelul este încărcat la cerere. Verifică memoria și jurnalul Ollama. Rulează diagnosticul `--self-test`. |
| Textul direct merge, dar modelul nu răspunde | Verifică furnizorul și modelul selectat. Compară cu `--self-test` pentru cele două modele incluse. |
| Sunetul lipsește | Testează **Rostește textul**, verifică volumul Windows și `Logs/piper.log`. |
| Textul extern nu ajunge la avatar | Verifică portul 8765, tokenul, endpointul și calculatorul de pe care este făcut apelul. |
| Fereastra se deschide, dar randarea are probleme | Consultă `Logs/avatar.log` și testează fereastra restaurată. `--cpu` schimbă doar inferența Ollama. |

Verificările efectuate pe calculatorul de dezvoltare au acoperit pornirea executabilului, generarea cu ambele modele, sinteza cu cele trei voci, concordanța dintre marcajele Piper și eșantioanele WAV, comenzi API și modurile ferestrei. `--self-test` verifică serviciile, în timp ce aspectul scenei și mișcarea facială se verifică în fereastra playerului. Compatibilitatea pe alt calculator Windows 11 x64 rămâne o probă de recepție. Răspunsurile generate depind de model și trebuie evaluate separat de reușita tehnică a redării.

### 10. Demonstrație propusă la susținere

1. Pornește pachetul prin `Start-VOX.cmd` și arată alegerea avatarului, a vocii și a modelului.
2. Selectează Adriana și rostește **„Mama pune mere. Bobi bea apă.”** Observă închiderea buzelor la M, B și P.
3. Repetă textul cu altă voce. Explică de ce timpii se recalculează pentru fiecare clip.
4. Pune o întrebare scurtă prin **Întreabă modelul**. Urmărește textul primit și rostirea frazelor.
5. Trimite un job `speak` prin API și arată ID-ul și rezultatul `done`.
6. Redimensionează fereastra, apoi testează **F11** și **Esc**.
7. Închide fereastra și explică de ce lansatorul deține ciclul de viață al serviciilor locale.

**Întrebări de pregătire.** De ce sunetul este ceasul principal al animației? Ce se întâmplă când Piper nu furnizează durate? De ce se grupează fragmentele în fraze? În ce fel diferă rigurile celor trei modele? Care este diferența dintre rezultatul tehnic al unei cereri și corectitudinea răspunsului modelului? Cum poate un formular online să trimită date fără a dezvălui tokenul în browser?

### 11. Credite și limite de distribuție

- Modelele vizuale Adriana, Toon3D și Witch provin din fișierele furnizate pentru proiect.
- Vocile **Lili High**, **Raluca High** și **Sanda High** sunt din proiectul `eduardem/piper-tts-romanian`(https://huggingface.co/eduardem/piper-tts-romanian).