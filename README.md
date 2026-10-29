#test
#test


int main() {
    setlocale(LC_ALL, "polish_poland");

    double wspol[LICZBA_WSPOL] = { 1.0, 1.0, 0.0, 1.0 };
    double xmin = -3.14;
    double xmax = 3.14;
    int n = 0;

    double *sygnal_bazowy = NULL;
    double *sygnal_szum = NULL;
    double *sygnal_filtr = NULL;

    int flaga_baza = 0;
    int flaga_szum = 0;
    int flaga_filtr = 0;

    int menu = -1;

    while (menu != 0) {
        printf("\n================ MENU PROJEKTU ================\n");
        printf("1. Pobierz parametry i wygeneruj sygnal podstawowy\n");
        printf("2. Wprowadz szum do sygnalu\n");
        printf("3. Przefiltruj sygnal (Medianowy / Srednia ruchoma)\n");
        printf("4. Zapisz wybrane dane do pliku CSV\n");
        printf("5. Wyswietl sygnal podstawowy w konsoli\n");
        printf("0. Zakoncz program\n");
        printf("Wybierz opcje: ");
        
        if (scanf("%d", &menu) != 1) {
            printf("Nieprawidlowy format wejscia!\n");
            while (getchar() != '\n'); // czyszczenie bufora
            continue;
        }

        switch (menu) {
            case 0:
                printf("Zakończenie programu.\n");
                break;

            case 1: {
                pobierzWspol(wspol);
                pobierzGranice(&xmin, &xmax);
                int nowe_n = pobierzLiczbeProbek();

                // Alokacja lub realokacja pamięci w przypadku zmiany liczby próbek
                double *tmp_baza = (double*)realloc(sygnal_bazowy, nowe_n * sizeof(double));
                double *tmp_szum = (double*)realloc(sygnal_szum, nowe_n * sizeof(double));
                double *tmp_filtr = (double*)realloc(sygnal_filtr, nowe_n * sizeof(double));

                if (tmp_baza == NULL || tmp_szum == NULL || tmp_filtr == NULL) {
                    printf("Blad alokacji pamieci!\n");
                    break;
                }

                sygnal_bazowy = tmp_baza;
                sygnal_szum = tmp_szum;
                sygnal_filtr = tmp_filtr;
                n = nowe_n;

                generujSygnal(wspol, xmin, xmax, sygnal_bazowy, n);
                flaga_baza = 1;
                flaga_szum = 0;   // reset przy nowym sygnale bazowym
                flaga_filtr = 0;
                printf("Sukces: Wygenerowano nowy sygnal podstawowy (%d probek).\n", n);
                break;
            }

            case 2: {
                if (!flaga_baza) {
                    printf("Blad: Najpierw wygeneruj sygnal podstawowy (Opcja 1)!\n");
                    break;
                }
                // Amplituda szumu jako 0.5 * |A|
                double max_amp = 0.5 * fabs(wspol[0]);
                if (max_amp == 0) max_amp = 0.5; // zabezpieczenie gdy A=0

                dodajSzum(sygnal_bazowy, sygnal_szum, n, max_amp);
                flaga_szum = 1;
                printf("Sukces: Wprowadzono szum do sygnalu.\n");
                break;
            }

            case 3: {
                if (!flaga_szum) {
                    printf("Blad: Najpierw wprowadz szum do sygnalu (Opcja 2), aby miec co filtrowac!\n");
                    break;
                }

                int wybor_filtru = 0;
                printf("Wybierz metode filtracji:\n");
                printf("1 - Filtr medianowy\n");
                printf("2 - Filtr sredniej ruchomej\n");
                printf("3 - Sekwencyjnie: Medianowy + Srednia ruchoma\n");
                printf("Wybór: ");
                scanf("%d", &wybor_filtru);

                if (wybor_filtru == 1) {
                    filtrMedianowy(sygnal_szum, sygnal_filtr, n);
                } else if (wybor_filtru == 2) {
                    filtrSredniejRuchomej(sygnal_szum, sygnal_filtr, n);
                } else if (wybor_filtru == 3) {
                    // Filtracja sekwencyjna (najpierw mediana, potem średnia)[cite: 7]
                    double *temp = (double*)malloc(n * sizeof(double));
                    if (temp != NULL) {
                        filtrMedianowy(sygnal_szum, temp, n);
                        filtrSredniejRuchomej(temp, sygnal_filtr, n);
                        free(temp);
                    } else {
                        filtrMedianowy(sygnal_szum, sygnal_filtr, n);
                    }
                } else {
                    printf("Nieprawidlowy wybor filtru.\n");
                    break;
                }

                flaga_filtr = 1;
                printf("Sukces: Przefiltrowano sygnal.\n");
                break;
            }

            case 4: {
                if (!flaga_baza) {
                    printf("Blad: Brak danych do zapisu. Wygeneruj najpierw sygnal!\n");
                    break;
                }

                int typ = 0;
                printf("Ktory sygnal chcesz zapisać do pliku?\n");
                printf("1 - Podstawowy\n");
                printf("2 - Zaszumiony %s\n", flaga_szum ? "" : "(brak danych)");
                printf("3 - Odfiltrowany %s\n", flaga_filtr ? "" : "(brak danych)");
                printf("Wybór: ");
                scanf("%d", &typ);

                const char *nazwa = "sygnal_podstawowy.csv";
                const double *dane = sygnal_bazowy;

                if (typ == 2) {
                    if (!flaga_szum) { printf("Brak zaszumionego sygnału!\n"); break; }
                    nazwa = "sygnal_zaszumiony.csv";
                    dane = sygnal_szum;
                } else if (typ == 3) {
                    if (!flaga_filtr) { printf("Brak odfiltrowanego sygnału!\n"); break; }
                    nazwa = "sygnal_odfiltrowany.csv";
                    dane = sygnal_filtr;
                }

                if (zapiszDoCSV(nazwa, dane, xmin, xmax, n)) {
                    printf("Sukces: Zapisano do pliku %s\n", nazwa);
                } else {
                    printf("Blad podczas zapisu pliku!\n");
                }
                break;
            }

            case 5: {
                if (!flaga_baza) {
                    printf("Blad: Brak wygenerowanego sygnału.\n");
                    break;
                }
                wyswietlSygnal(sygnal_bazowy, n);
                break;
            }

            default:
                printf("Nieprawidlowa opcja w menu!\n");
        }
    }

    // Zwolnienie dynamicznej pamięci przed wyjściem
    free(sygnal_bazowy);
    free(sygnal_szum);
    free(sygnal_filtr);

    return 0;
}
