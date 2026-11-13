# Przegląd komunikacji — Restaurant Management System (Checkpoint 3)

## Podstawa

Przegląd dotyczy `restaurant-api/REST_API_ENDPOINTS.md` i `restaurant-api/openapi-clean.yaml`, udostępnionych wcześniej w tej rozmowie. Diagramy są projektem komunikacji na podstawie tych plików — nie dokumentacją uruchomionego backendu. W `coverage.csv` znajduje się mapowanie 38 identyfikatorów operacji na 10 kart diagramu `.drawio`.

## Błędy i niespójności wymagające decyzji

| Ważność | Problem | Gdzie | Co proponuję |
|---|---|---|---|
| Krytyczna | Nie określono, jak gość udowadnia prawo do odczytu/anulowania własnego zamówienia lub rezerwacji. W YAML dostęp jest dopuszczony bez Bearera (`security: [{}, {bearerAuth: []}]`), ale nie widać zdefiniowanego lookup tokenu. Sam `orderId`/`reservationId` nie powinien stanowić dowodu uprawnienia. | `GET /orders/{orderId}`, `GET /orders/{orderId}/payment`, `POST /orders/{orderId}/cancel`, `GET /reservations/{reservationId}`, `POST /reservations/{reservationId}/cancel` | Dodać bezpieczny token dostępu gościa (nagłówek lub bezpieczny link), ograniczyć uprawnienia albo wymagać konta — konsekwentnie w YAML i diagramach. |
| Krytyczna | Brakuje jednoznacznego sposobu utworzenia obiektu `Delivery` po uzyskaniu statusu `READY`. Jest API do przypisania i zmiany statusu, ale nie ma opisu powstania dostawy. | Zamówienia i dostawy | Zdefiniować automatyczne utworzenie dostawy jako operację backendu podczas zmiany stanu zamówienia; nie musi być publicznym endpointem. |
| Wysoka | Przepływ płatności jest częściowo nieokreślony: `POST /orders` tworzy `PENDING_PAYMENT`, webhook zatwierdza transakcję, lecz nie ma szczegółowej specyfikacji inicjowania płatności u operatora, wygasania, zwrotów i uzgadniania nieznanego wyniku. | `ORD-01`, `PAY-02`; `Order.paymentStatus` | Uzgodnić jeden scenariusz asynchroniczny, unikalny `eventId`, idempotencję, retry oraz stan po timeout; doprecyzować schemat provider request/response. |
| Wysoka | Role i zakres uprawnień są opisane tylko własnym polem `x-required-roles`; walidator OpenAPI go nie interpretuje. Bearer token nie gwarantuje, że użytkownik ma rolę managera/ownera ani że jest właścicielem konkretnego zamówienia. | Chronione endpointy OpenAPI | Opisać macierz autoryzacji, rozstrzygnąć przyznawanie roli ADMIN oraz kontrolę ownership po stronie backendu. |
| Wysoka | Główny schemat `OrderCreate` wymaga `paymentMethodToken`, natomiast API nie dokumentuje interakcji frontend ↔ operator płatności, dzięki której ten token powstaje. | `POST /orders` / Payment Provider | Dodać opis tokenizacji w zewnętrznym providerze i diagramie lub przyjąć inny, spójny mechanizm płatności. |
| Wysoka | Nie określono źródła danych o kosztach, zwrotach i anulowanych zamówieniach; samo API raportowania nie wystarczy do policzenia wiarygodnego zysku. | `/reports/financial/*` | Wskazać moduł i sposób wprowadzania kosztów oraz reguły rachunkowe. |
| Średnia | Nie podano długości rezerwacji, strefy czasowej i warunku nakładania się rezerwacji. Sprawdzenie `GET /reservations/availability` może się zdezaktualizować przed zapisem. | `RES-01`, `RES-02` | Ustalić reguły przedziałów czasowych oraz transakcyjne sprawdzenie i zapis w `POST /reservations` (konflikt 409). |
| Średnia | Brak jednoznacznej synchronizacji statusu `Delivery.DELIVERED` i `Order.COMPLETED`; ryzyko sprzecznych historii. | `DEL-06`, `ORD-05` i enumy | Ustalić jeden proces aktualizacji zakończenia dostawy i zamówienia, najlepiej w transakcji/zdarzeniu. |
| Średnia | Lista odpowiedzi błędów jest niemal identyczna dla wszystkich endpointów, również np. publicznych GET; zawiera statusy, które mogą nie mieć sensownej przyczyny dla konkretnej operacji. Dodatkowo dla błędnego logowania brakuje dokumentacji `401` (jest m.in. `404/409`). | `openapi-clean.yaml` `responses` | Zawęzić listy statusów per operacja; dla `POST /auth/login` opisać `401` i nie ujawniać istnienia konta. |
| Średnia | Pobieranie konkretnych rezerwacji po ścieżce `/reservations/{reservationId}` współistnieje z `/reservations/trends`, a menu posiada publiczne `/menu/{dishId}` i managerskie `/menu/items/{dishId}`. To nie musi być błąd routingu, ale łatwo o niejednoznaczną dokumentację i konfigurację. | Paths | Zachować pierwszeństwo ścieżek statycznych; rozważyć ujednolicenie zasobów menu i wydzielenie raportów trendów pod `/reports/reservations/trends`. |
| Niska | Eksport CSV/PDF ma jeden ogólny typ odpowiedzi `application/octet-stream`, choć opis mówi o `text/csv` lub `application/pdf`. | `GET /reports/financial/export` | W specyfikacji zadeklarować oba formaty odpowiedzi. |
| Niska | `PATCH /menu/items/{dishId}` pozwala na pusty obiekt `DishPatch` (brak wymogu minimum 1 pola). | `DishPatch` | Użyć `minProperties: 1` i zablokować nieznane pola, jeśli taką politykę wybierzecie. |

## Uproszczenia, które nie zmieniają funkcjonalności

1. **10 diagramów zamiast 38** — grupowanie według procesów, ale każdy identyfikator endpointu ma odwzorowanie w `coverage.csv`.
2. **Jeden wspólny schemat błędów** (`Error`) i jednolity format odpowiedzi listowych (`items`, `nextCursor`), bez dublowania 38 kopii tych samych definicji.
3. **Dwie granice API**: publiczny REST `/api/v1` i zewnętrzne interfejsy operatorów płatności/powiadomień/map. Zewnętrznych adresów nie należy wymyślać, dopóki nie wybierzecie dostawcy.
4. **Oddzielić status zamówienia od statusu dostawy i płatności**; nie duplikować pełnego stanu każdego podsystemu we wszystkich komponentach.
5. **Nie tworzyć REST CRUD dla wewnętrznych zdarzeń** takich jak automatyczne utworzenie dostawy — wystarczy opisana operacja domenowa backendu.
6. **Ograniczyć logikę diagramów do komunikacji**: szczegółowe sprawdzanie warunków biznesowych pozostawić Activity / State Diagrams; tutaj pokazać żądania, odpowiedzi i kluczowe warianty błędów.

## Jak czytać diagramy

- Każda zakładka w `.drawio` to inny proces; wszystkie prostokąty, linie, podpisy i notatki są edytowalne.
- Strzałka ciągła oznacza żądanie lub wywołanie; przerywana oznacza odpowiedź.
- Pomarańczowy wiersz `alt [...]` i następna strzałka to ważny wariant błędu.
- Komunikacja z `Database`, `Auth/RBAC`, `Notification Service`, `Payment Provider` i `Routing Provider` przedstawia logikę/proponowane granice komponentów — nie jest listą publicznych endpointów REST.
- Diagramy obejmują wszystkie 38 operacji wymienionych w inwentarzu, **ale nie stanowią jeszcze pełnej formalnej specyfikacji wszystkich interfejsów wewnętrznych i zewnętrznych**, której wymaga wersja końcowa.

## Kontrola przed oddaniem

- Uzgodnić pięć decyzji o najwyższym priorytecie (guest token, powstanie dostawy, inicjowanie płatności, role, koszty finansowe).
- Uzupełnić/zmienić `openapi.yaml` zgodnie z decyzjami.
- Zaktualizować podpisy komunikatów w `.drawio`, jeśli zmienią się nazwy pól lub endpointów.
- Wyeksportować strony diagramu do **PDF** z diagrams.net i dołączyć przykłady schematów JSON do oddawanego PDF-a; źródłowy `.drawio` zachować w repozytorium.
