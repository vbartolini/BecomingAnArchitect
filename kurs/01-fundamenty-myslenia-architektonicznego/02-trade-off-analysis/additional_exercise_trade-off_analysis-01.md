**Problem:** Warstwa dostępu do danych w module raportowym ma być wydajna przy kilku ciężkich zapytaniach analitycznych i jednocześnie łatwa w utrzymaniu przez resztę zespołu.

**Kryteria (w kolejności priorytetu):** (1) wydajność zapytań read-heavy, (2) czytelność/utrzymywalność, (3) kompetencje zespołu, (4) spójność z resztą kodu

| Kryterium | A: Użycie procedur składowanych wywoływanych z poziomu ORM | B: Dapper - micro-ORM | C: Moduł korzystający z pełnego mapowania EF | 
|---|---|---|---|
|wydajność zapytań read-heavy | ++ możliwość tweakowania ciężkich raportów | + prawie tak wydajny jak raw SQL | -- ORM potrafi ignorować wydajność|
|czytelność/utrzymywalność | + lepsza, bo dotyczy tylko raportów, w których mogą być "zaszyte" pewne corner-case'y | + kod SQL utrzymywany w solucji .NET |- trudność w odseparowaniu raportów od standardowej logiki |
|kompetencje zespołu | 0 wymaga znajomości T-SQL, co nie jest standardem w dzisiejszych czasach wśród programistów .NET | + wymaga znajomości czystego SQL, który mapuje wynik na obiekty w .NET | + nie wymaga znajomości SQL, chociaż warto byłoby jednak mieć kogoś, kto byłby w stanie przechwycić query i ją zweryfikować pod kątem wydajności |
|spójność z resztą kodu | --logika domenowa w dwóch miejscach|++ w pełni spójny z resztą kodu| ++logika domenowa w jednym miejscu|

**Odwracalność:** Wszystkie trzy opcje to two-way door — kod zamknięty w module raportowym, podmiana to godziny, nic na zewnątrz nie zależy od sposobu odczytu (callerzy dostają DTO). Decyzja lekka → wybieram szybko, bez ciężkiej analizy. Door zatrzasnąłby się dopiero, gdyby ktoś z zewnątrz zaczął wołać te procedury bezpośrednio — wtedy awans do one-way.

**Rekomendacja** Opcja B. Świadomie rezygnujemy z ekstremalnej wydajności czystych procedur na rzecz utrzymywalności w jednym repozytorium i możliwości zaaplikowania wspólnej logiki. Jeżeli Dapper okaże się niewydajny w p95, wtedy dla tego raportu rozważę procedurę składowaną.


