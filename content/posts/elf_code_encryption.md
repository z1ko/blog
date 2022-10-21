---
title: "ELF code encryption"
author: "Filippo Ziche"
description: "In questo post vedremo come creare un eseguibile con del codice criptato."
date: 2022-10-21
draft: false
---

# L'idea

Per questo progetto volevo creare una libreria e dei tools per facilitare la creazione di codice criptato. L'idea è quella di produrre un eseguibile diviso in due sezioni, un loader e una regione criptata.
Il loader ha il compito di decriptare la seconda regione dopo aver, ad esempio, superato un controllo sulla licenza.

# Setup

```c
#define POLYV_IMPLEMENTATION
#include <polyv.h>

// Funzione che verra automaticamente criptata nell'eseguibile finale
void POLYV_ENCRYPTED encrypted_function() { /* ... */ }

// Funzione normale, non verrà criptata
void normal_function() { /* ... */ }
```

Definiamo delle macro per semplificarci la vita, tutte le funzioni con il nostro attributo POLYV\_ENCRYPTED verranno inserite nella regione segreta dal linker 

```c
#define POLYV_ENCRYPTED_SECTION ".etext"
#define POLYV_ENCRYPTED __attribute__ ((section (POLYV_ENCRYPTED_SECTION)))
```

Ma cosa succede se non abbiamo una sola unità di compilazione? E come facciamo a capire l'inizio e la fine della regione nascosta? Entrambi questi quesiti possono essere risolti creando uno script per il linker tutto nostro, un'antica arte che solamente gli sviluppatori di firmware conoscono.

```ld
SECTIONS {
    OVERLAY : {
        .etext
        {
            . = ALIGN(4); 
            *(.etext)
        }
    }
    .ekey : { *(.ekey) }

} INSERT AFTER .text;
```

Se chiamiamo questo script encrypted.ld, possiamo specificare usarlo durante la compilazione con la flag -Tencrypted.ld.

# Implementazione

Ora possiamo scrivere la funzione del loader che decripterà la regione nascosta.

```c
// Simboli generati dal linker per individuare l'inizio e la fine della regione criptata
extern char __load_start_etext, __load_stop_etext;

// Decripta la regione critica
void POLYV_LOADER polyv_self_sxor(char* key, size_t key_size) {

    size_t section_beg = (size_t)&__load_start_etext;
    size_t section_end = (size_t)&__load_stop_etext;

    const size_t page_size = getpagesize();
    const size_t section_delta = section_end - section_beg;
    size_t section_size = section_delta;

    if (section_size % page_size != 0)
        section_size = section_delta + (section_delta % page_size);

    // Enable writing of the section
    void* section_page = (void*)(section_beg - (section_beg % page_size));
    mprotect(section_page, section_size, PROT_EXEC | PROT_WRITE | PROT_READ);

    // Apply symmetric XOR with provided key
    char* section = (char*)(section_beg);
    polyv_sxor(section, section_delta, key, key_size);

    // Disable writing of the section
    mprotect(section_page, section_size, PROT_EXEC | PROT_READ);
}
```
# Conclusione

Grazie! Alla prossima.
FZ.

