# Whisper large-v3 Transkription af Teams interview

Workflow til dansk transskription af Teams-interviews: **Whisper large-v3 til ordlyden og Teams-VTT til talernavne**. Dokumentet fastholder også erfaringerne fra et lille eksperiment den 26. oktober 2026, så modelvalget og de afprøvede veje ikke skal undersøges forfra.

Repoet indeholder kun dokumentation. Interviewoptagelser, transskriptioner, personnavne, modelvægte og adgangsnøgler indgår ikke.

## Beslutningen

Vi vælger **Whisper large-v3 som grundlag for renskrift**. Brugeren vurderede den direkte sammenligning med Hviske v3 conversation og valgte Whisper som klart bedst i det konkrete materiale.

Teams-transskriptionen bruges til at identificere talerne, ikke som facit for ordlyden. Whispers tekst skal fortsat kontrolleres for fagudtryk, navne og uklare passager.

## Det valgte workflow

### 1. Find optagelsen og Teams-transskriptionen

Brug Teams-optagelsen, typisk en MP4, og den tilhørende VTT-fil. Billedsiden er ikke nødvendig for selve talegenkendelsen. Skærmdeling kan dog indeholde faglig kontekst, som hjælper ved efterkontrol.

VTT-filen indeholder tidsstempler og talernavne, eksempelvis som `<v Talernavn>…</v>`. Workflowet anvender VTT; Word-eksporten var til rådighed i eksperimentet, men blev ikke anvendt til taleropdelingen.

### 2. Udtræk lydsporet

Udtræk lyd til WAV med én kanal, 16.000 Hz og 16-bit PCM. Bevar tidslinjen fra optagelsens begyndelse. Hvis optagelsen beskæres eller stilhed fjernes permanent, skal denne ændring medregnes ved sammenkoblingen med VTT.

Vi brugte **PyAV**, der indeholder de nødvendige FFmpeg-biblioteker. Det krævede derfor ikke en særskilt FFmpeg-installation. Originaloptagelsen blev ikke ændret.

### 3. Transskriber med Whisper large-v3

Kør modellen lokalt med **faster-whisper**, som bruger CTranslate2. Den afprøvede opsætning var:

| Indstilling | Værdi |
|---|---|
| Model | `large-v3` |
| Sprog | Dansk (`da`) |
| Opgave | Transskription (`transcribe`) |
| Enhed | CPU |
| Beregningstype | `int8` |
| CPU-tråde | 8 |
| Beam size | 5 |
| Taleaktivitetsdetektion, VAD | Til |
| Ordtidsstempler | Til |
| Kontekst fra foregående del | Til |
| Temperaturgenforsøg | Bibliotekets standardindstilling |
| Fagordsliste / initial prompt | Ingen i eksperimentet |

Gem både rå tekst og et struktureret JSON-resultat med segmenter, ord og tidsstempler. Gem også indstillingerne sammen med resultatet. Ordtidsstemplerne bruges i næste trin.

Modellen kan hentes første gang og derefter køres lokalt. Interviewlyden blev ikke sendt til en ekstern transskriptionstjeneste i vores forsøg.

### 4. Knyt talernavne til Whispers tekst

Læs VTT-filen med **webvtt-py**. Match Whispers ord mod VTT-teksten med Python-standardbibliotekets **difflib**, og brug matchene som tekstankre.

1. Normaliser ord til matchning, herunder store/små bogstaver og tegnsætning. Bevar den oprindelige Whisper-tekst til output.
2. Find sammenhængende ordmatch mellem Whisper og VTT.
3. Beregn en typisk tidsforskydning fra egnede tekstankre, eksempelvis med medianen. Genberegn den for hvert interview; brug ikke en fast forskydning fra dette forsøg.
4. Brug tekstankre og tidsmæssigt overlap til at tildele VTT-talernavne til Whisper-ordene. Sammenhængende tekstmatch hjælper med at skelne hovedtaleren fra overlappende, korte indskud.
5. Saml ordene i læsevenlige taleture med navn og tidsinterval.
6. Markér tvetydige tildelinger med `[Taler usikker]`. Brug `Ukendt taler`, når grundlaget er utilstrækkeligt.
7. Kontrollér maskinelt, at alle Whisper-ord er bevaret i samme rækkefølge, og at navnene stammer fra VTT.

Dette er overførsel af eksisterende talerinformation, **ikke akustisk taleridentifikation**. Fejl i Teams-talernavne kan følge med. En manglende usikkerhedsmarkering er heller ikke garanti for en korrekt tildeling.

Korte svar, som Whisper har udeladt, skal ikke indsættes fra Teams, som om de var transskriberet af Whisper. En eventuel tilføjelse kræver særskilt kontrol mod lyden.

### 5. Foretag efterkontrol og renskrift

Kontrollér især usikre talerskift, korte indskud, overlappende tale, navne og fagudtryk. I vores materiale gav blandt andet LER, MicroStation og et vejnavn problemer.

Bevar rå transskription og taleropdelt version som dokumentation. Foretag faglige rettelser i en særskilt renskrift. Markér uforståelige passager med tidsangivelse frem for at gætte. Beslut særskilt, om fyldord og gentagelser skal bevares eller renses væk; det afhænger af interviewets formål.

Det forventede resultat er en transskription med talernavne, tidsangivelser og synlige usikkerheder, suppleret med JSON til sporbarhed.

## Eksperimentet — hvad vi allerede har undersøgt

### Materiale og formål

Vi brugte de samme første **300 sekunder** af et dansk Teams-interview på cirka 37 minutter. Interviewet handlede om ledningskoordinering og indeholdt fagudtryk, gentagelser og korte indskud. Den oprindelige Teams-VTT var til rådighed.

Formålet var at vælge et praktisk grundlag for transskription og undersøge, om Teams-talernavne kunne kobles til den nye tekst. Femminuttersudsnittet var forsøgsdesignet; prøvekørsler og modelsammenligning er **ikke et trin i det valgte workflow**.

Computeren havde AMD Ryzen 7 PRO 7730U, cirka 16 GB RAM og ingen NVIDIA-GPU. Beregningen foregik på CPU.

### Modeller og gennemførte kørsler

| | Whisper large-v3 | Hviske v3 conversation |
|---|---|---|
| Modelgrundlag | Whisper large-v3 via faster-whisper | `syvai/hviske-v3-conversation`, lokalt konverteret med CTranslate2 |
| Fælles indstillinger | Dansk, CPU, INT8, beam size 5, VAD | Dansk, CPU, INT8, beam size 5, VAD |
| Kørselsform | Sekventiel | Batchstørrelse 2 |
| Tidsstempler | Ordtidsstempler | Uden genererede tidsstempler; omtrentlige blokintervaller |
| Kontekst fra foregående del | Til | Fra |
| Temperatur | Standardgenforsøg | Fast 0 |
| Transskriptionstid | 852,8 sekunder, cirka 14,2 minutter | 406,4 sekunder, cirka 6,8 minutter |

Tiderne omfatter selve transskriptionen, ikke installation, download, indlæsning eller modelkonvertering. Opsætningerne var forskellige: forsøget dokumenterer derfor hverken en generel hastighedsfaktor eller en kontrolleret sammenligning af modelnøjagtighed.

### Hvad vi så i teksterne

- **Whisper** gav gennemgående mere tegnsætning og en mere læsevenlig tekst.
- **Hviske** bevarede flere fyldord og gentagelser, men havde også uklare formuleringer i tekniske passager.
- Ved omtalen af LER skrev Whisper “søges læger”, mens Hviske skrev “søges ler”.
- Begge havde problemer med vejnavnet og programnavnet MicroStation. Hviske gengav blandt andet programnavnet som “Microsoft”.
- I en anden passage skrev Whisper “Her får jeg den store polygon ind”, mens Hviske skrev “her får jeg den store på det gode mand”.

Brugeren foretrak på den baggrund Whisper. Det er beslutningen for dette materiale, ikke en påstand om, at modellen altid er bedst til danske interviews.

Der blev ikke udarbejdet en manuelt verificeret referencetekst eller beregnet Word Error Rate. Modelteksterne blev sammenlignet som rå resultater uden fuld manuel lydkontrol. Teams-teksten var heller ikke et facit.

### Afprøvede Hviske-veje, som ikke skal gentages som standard

1. **Kørsel med genererede tidsstempler:** afbrudt efter ustabile spring i tidslinjen og mistanke om udeladelser. Det viste problemer i denne opsætning, ikke en dokumenteret begrænsning i alle mulige Hviske-opsætninger.
2. **Sekventiel tekstkørsel uden tidsstempler:** afbrudt, fordi den arbejdede meget langsomt på en efterfølgende lydblok. Årsagen blev ikke endeligt fastslået.
3. **Batchkørsel uden genererede tidsstempler:** gennemførte alle fem minutter med indstillingerne i tabellen ovenfor. Det er denne færdige tekst, sammenligningen anvendte.

Hviske krævede accept af adgangsbetingelser på Hugging Face. Modellen blev til sidst hentet manuelt gennem browseren. Et adgangstoken er relevant for programmatisk download, men var ikke nødvendigt for brugerens manuelle download efter login og godkendelse.

### Resultat af taleropdelingen

Whispers resultat blev efterfølgende koblet til Teams-VTT med ordmatch og tidsmæssigt overlap:

- 45 tekstankre gav en medianforskydning på **1,495 sekunder**, med VTT senere end Whisper.
- Medianen af de absolutte afvigelser var cirka **0,49 sekund**. Forskydningen er en tilpasning af automatiske tidsangivelser, ikke en måling af en fysisk lydforsinkelse.
- Alle **727 Whisper-ordposter** blev bevaret i samme rækkefølge.
- Resultatet blev samlet i **19 viste afsnit**.
- **Tre passager** blev markeret med usikker talertildeling; én af disse havde ukendt taler.
- Brugeren vurderede den taleropdelte prøve positivt.

Det viser, at VTT-talernavne er et brugbart supplement til Whisper i dette materiale. Det dokumenterer ikke fejlfri taleropdeling. Word-eksporten blev ikke nødvendig, og resten af interviewet blev ikke transskriberet som del af eksperimentet.

## Værktøjer og reproducerbarhed

De centrale versioner i det gennemførte arbejde var:

| Værktøj | Version | Anvendelse |
|---|---|---|
| Python | 3.11 | Lokalt miljø |
| faster-whisper | 1.2.1 | Transskription |
| CTranslate2 | 4.8.4 | Lokal modelkørsel og Hviske-konvertering |
| PyAV | 18.1.0 | Lydudtræk |
| webvtt-py | 0.6.0 | VTT-indlæsning |
| difflib | Python-standardbibliotek | Tekstmatch til talernavne |

Transformers og PyTorch blev også installeret til Hviske-konverteringen. De er ikke nødvendige for det valgte Whisper-workflow med faster-whisper.

Repoet indeholder ikke eksekverbare scripts, et låst installationsmiljø eller forsøgsdata. Det fastholder metode, indstillinger, begrænsninger og beslutninger; dokumentationen alene giver derfor ikke en identisk genkørsel med ét klik.

## Praktiske rammer for videre arbejde

Arbejd på separate outputfiler og bevar originalerne. I det oprindelige arbejdsmiljø placeres testdata og midlertidige filer under `%USERPROFILE%\Downloads\_claude_test\`. Produktion og persondata holdes uden for dette repo.

Inden commit og push skal repoets indhold gennemgås. `.gitignore` reducerer risikoen for utilsigtet tilføjelse af interviewdata, men erstatter ikke denne kontrol.

## Referencer

- [faster-whisper](https://github.com/SYSTRAN/faster-whisper)
- [Whisper large-v3](https://huggingface.co/openai/whisper-large-v3)
- [Hviske v3 conversation](https://huggingface.co/syvai/hviske-v3-conversation)
- [webvtt-py](https://github.com/glut23/webvtt-py)
