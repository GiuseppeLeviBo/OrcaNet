Valutazione approfondita del progetto legacy OrcaNet e piano di ammodernamento
=============================================================================

Obiettivo del documento
-----------------------
Questo documento descrive in modo strutturato:

- funzionalità principali di OrcaNet;
- modalità d'uso operative (CLI e API Python);
- stato tecnico "legacy" attuale;
- piano di ammodernamento verso una piattaforma moderna (CUDA 12.x + Python moderno).


Executive summary
-----------------
OrcaNet è un framework orientato al training, validazione e inferenza di modelli deep learning su dataset HDF5 multi-file per il dominio astroparticellare (KM3NeT). Il cuore applicativo è la classe ``Organizer``, che coordina pipeline dati, lifecycle del modello, logging e output di training.

La base tecnologica è oggi ancorata a TensorFlow 2.5.x con dipendenze coeve (TensorFlow Addons 0.12, TensorFlow Probability 0.12), tipicamente legate a stack CUDA non recenti. L'ammodernamento richiede un percorso incrementale con:

1. stabilizzazione test e baseline riproducibile;
2. salto intermedio compatibilità (TensorFlow moderno mantenendo API storiche);
3. adozione stack target (Python 3.11/3.12, CUDA 12.x);
4. hardening operativo (CI matrix, container moderni, test di regressione scientifica).


1) Funzionalità del progetto
----------------------------

1.1 Workflow end-to-end
~~~~~~~~~~~~~~~~~~~~~~~
OrcaNet copre un ciclo completo:

- avvio/continuazione training;
- validazione periodica;
- salvataggio checkpoint e log;
- predizione su validation set;
- inferenza su set dedicati;
- esportazione risultati in HDF5.

I subcomandi principali della CLI sono ``train``, ``predict``, ``inference``, ``summarize`` e ``version``.


1.2 Gestione esperimenti e stato
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
La classe ``Organizer`` centralizza:

- configurazione runtime (tramite ``Configuration``);
- accesso I/O a file di training/val/inference;
- storico e stato del training;
- resume automatico da checkpoint;
- orchestrazione train/validate/predict/inference.

Questo approccio rende la pipeline robusta per dataset voluminosi e training lunghi.


1.3 Data pipeline HDF5 multi-file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
La pipeline gestisce dataset distribuiti su più file HDF5, con:

- generatori batch-wise;
- supporto a modifier di sample/label/dataset;
- normalizzazione (es. zero-center);
- conversione a ``tf.data.Dataset`` in alcuni percorsi.

È una caratteristica chiave per carichi tipici HPC/scientifici.


1.4 Model building configurabile via TOML
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Il ``ModelBuilder`` costruisce modelli da file TOML:

- definizione layer/block modulari;
- mapping input in base ai dataset disponibili;
- compilazione con loss e metric custom;
- integrazione con custom objects e modifier.

Questo riduce il codice imperative per esperimento e favorisce riproducibilità.


1.5 Logging, reportistica e visualizzazione
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Il progetto include:

- log incrementali di training;
- plot di summary e utilità di visualizzazione;
- comandi CLI per riepilogo andamento training.

In contesto scientifico, questi artefatti sono essenziali per audit e confronto run.


2) Modalità d'uso
-----------------

2.1 Uso via CLI (consigliato per operatività)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Struttura standard di run directory:

- ``list.toml``: definizione file train/val/inference;
- ``config.toml``: opzioni training;
- ``model.toml``: architettura/compile del modello.

Flusso minimo:

1. creare directory run;
2. inserire i tre TOML;
3. lanciare ``orcanet train <run_dir>``;
4. monitorare output/plot;
5. eseguire ``orcanet predict <run_dir>`` o ``orcanet inference <run_dir>``.


2.2 Uso via API Python (integrazione avanzata)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Pattern operativo:

1. istanziare ``Organizer(output_folder, list_file, config_file)``;
2. costruire modello (manualmente o con ``ModelBuilder``);
3. chiamare ``train_and_validate`` / ``predict`` / ``inference``.

Questa modalità è utile quando si vogliono automatizzare sweep, pipeline esterne o integrazioni custom.


2.3 Entry point legacy/deprecati
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Sono presenti script contrib ``orcatrain``/``orcapred`` marcati come deprecati a favore della CLI unificata ``orcanet``. Nel piano di ammodernamento è opportuno definire una strategia di rimozione graduale (con finestra di compatibilità documentata).


3) Diagnosi tecnica legacy (stato attuale)
------------------------------------------

3.1 Stack dipendenze e runtime
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Elementi indicativi di legacy:

- ``tensorflow==2.5.*``;
- ``tensorflow_addons==0.12.*``;
- ``tensorflow_probability==0.12.*``;
- immagine Docker basata su ``tensorflow/tensorflow:2.5.0-gpu``.

Questa baseline è storicamente valida ma oggi distante dalle versioni Python/CUDA più recenti.


3.2 Rischi principali
~~~~~~~~~~~~~~~~~~~~~
Rischi tecnici e operativi da mitigare:

- compatibilità pacchetti con Python moderni;
- disallineamento CUDA/cuDNN rispetto a release attuali;
- regressioni silenziose su training numerico durante migration;
- obsolescenza di dipendenze accessorie (es. Addons in alcune API);
- aumento costo manutenzione per build/container legacy.


3.3 Opportunità
~~~~~~~~~~~~~~~
L'architettura è ben separata (core, model builder, backend, I/O, utilities) e questo favorisce un refactor incrementale con basso rischio di blocco totale.


4) Target di ammodernamento proposto
------------------------------------

4.1 Target tecnologico consigliato
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Target pragmatico (2026-ready):

- Python: 3.11 come default (valutare 3.12 su CI come experimental/stable);
- TensorFlow: ramo 2.16+ (in base a compatibilità dipendenze scientifiche);
- CUDA: 12.2/12.4 (scegliere una minor unificata in base a cluster/driver);
- cuDNN: versione allineata alla release TensorFlow scelta;
- packaging: ``pyproject.toml`` (PEP 517/518) + metadata moderni.


4.2 Principi guida
~~~~~~~~~~~~~~~~~~

- migrazione a step, mai "big-bang";
- mantenere sempre un ramo stabile rilasciabile;
- introdurre test di regressione numerica oltre ai test funzionali;
- usare container pin-nati per riproducibilità.


5) Piano di ammodernamento (roadmap)
------------------------------------

Fase 0 — Baseline e inventario (1-2 settimane)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- congelare ambiente legacy funzionante (requirements lock + container digest);
- catalogare use-case scientifici critici (training, resume, predict, inference);
- definire dataset di regressione ridotti ma rappresentativi;
- raccogliere metriche baseline (loss curve, tempi epoch, throughput, memoria GPU).

Deliverable:

- report baseline prestazionale/numerica;
- "golden artifacts" (log e output attesi).


Fase 1 — Hardening test e CI (1-2 settimane)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- estendere test automatici su:
  - parser CLI,
  - pipeline HDF5,
  - model building TOML,
  - resume training,
  - predict/inference output schema;
- creare matrice CI minima (Python 3.10 legacy + 3.11 target);
- separare test "fast" da test "integration".

Deliverable:

- pipeline CI affidabile;
- soglie di accettazione regressione.


Fase 2 — Aggiornamento packaging & qualità codice (1 settimana)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- introdurre ``pyproject.toml`` mantenendo compatibilità installazione;
- centralizzare configurazioni tool (pytest, lint, format);
- aggiornare metadata progetto e classificatori Python supportati;
- rimuovere gradualmente entrypoint deprecati o marcarli con piano EOL.

Deliverable:

- build/install moderni;
- policy supporto versioni chiara.


Fase 3 — Migrazione TensorFlow compatibile (2-4 settimane)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- aggiornare TensorFlow a release ponte moderna;
- sostituire API deprecate dove necessario (optimizer attrs, callback semantics, ecc.);
- verificare compatibilità ``tensorflow_probability`` e ``tensorflow_addons``:
  - se Addons non necessario, ridurne uso;
  - dove possibile sostituire con primitive core Keras/TensorFlow;
- consolidare uso ``tf.data`` nei punti in cui oggi convivono generatori legacy.

Deliverable:

- suite test verde su stack TF moderno;
- diff numerico entro tolleranze definite.


Fase 4 — Allineamento CUDA 12.x e container runtime (1-2 settimane)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- creare nuove immagini Docker/Singularity con CUDA 12.x + cuDNN coerente;
- validare su GPU target (A100/H100/L40 o hardware effettivo cluster);
- benchmark comparativi vs baseline legacy.

Deliverable:

- container production-ready;
- report performance e stabilità GPU.


Fase 5 — Regressione scientifica e rilascio controllato (1-2 settimane)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- riesecuzione pipeline scientifiche di riferimento;
- confronto distribuzioni output e metriche finali;
- rilascio candidato (RC) con changelog migrazione;
- guida upgrade utenti (breaking changes + fallback).

Deliverable:

- release modernizzata;
- playbook migrazione utenti.


6) Backlog tecnico consigliato (priorità)
-----------------------------------------

Alta priorità
~~~~~~~~~~~~~

1. Aggiornare dipendenze core e bloccare versioni compatibili.
2. Implementare test di regressione numerica su mini-dataset noto.
3. Modernizzare container GPU e documentare matrix driver/CUDA.

Media priorità
~~~~~~~~~~~~~~

1. Ridurre dipendenza da componenti contrib deprecati.
2. Migliorare osservabilità (log strutturato, metriche runtime).
3. Consolidare API pubblica e segnare chiaramente API interne.

Bassa priorità
~~~~~~~~~~~~~~

1. Migrazione graduale documentazione a stile "how-to" + "reference".
2. Esempi notebook moderni con pipeline riproducibile.


7) Strategia di compatibilità e rischio
---------------------------------------

Proposta compatibilità:

- mantenere un branch ``legacy-lts`` per bugfix critici a breve termine;
- branch ``main`` orientato al nuovo stack;
- deprecazioni comunicate su due release prima della rimozione.

Mitigazioni rischio:

- canary release su subset utenti;
- fallback container legacy sempre disponibile durante transizione;
- gating merge con test GPU + regressione numerica.


8) Indicazioni operative immediate (prossimi 30 giorni)
--------------------------------------------------------

Settimana 1
~~~~~~~~~~~

- congelare baseline legacy;
- definire mini-dataset di regressione;
- impostare CI base.

Settimana 2
~~~~~~~~~~~

- aggiornare packaging;
- avviare migrazione TF "ponte" in feature branch.

Settimane 3-4
~~~~~~~~~~~~~

- completare correzioni compatibilità API;
- primi benchmark su container CUDA 12.x;
- preparare documento "breaking changes".


Conclusione
-----------
OrcaNet ha fondamenta solide sul piano funzionale (orchestrazione training scientifico, pipeline HDF5, configurazione TOML, resume e inferenza), ma richiede un aggiornamento infrastrutturale deciso per restare sostenibile su stack moderni Python/CUDA.

La roadmap proposta minimizza il rischio scientifico perché separa chiaramente: baseline, migrazione tecnica, validazione numerica e rilascio controllato.
