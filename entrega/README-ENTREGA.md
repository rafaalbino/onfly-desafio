# Entrega Projeto Onfly
---
### Checklist pré investigação

* \[✅] Ambiente configurado
* \[✅] O CLI roda com Python 3.11 (`$env:PYTHONPATH="src"; python -m onfly_report ...`).
* \[✅] O comando de execução gera `output/report.json` sem erro.
* \[❌] O relatório segue o contrato acima.
* \[❌] `pytest -q` passa com os testes fornecidos.
* \[❌] `entrega/README-ENTREGA.md` preenchido.
* \[❌] Sessão + declaração apenas no caminho com IA; investigação no README-ENTREGA nos dois caminhos.
* \[❌] Repositório público no GitHub com o projeto e o link enviado.

---

## Investigando por partes:
### Resumo da Saída do relatório:

|Campo|Valor|
|-|-|
|`total\_reservations`|12|
|`confirmed\_reservations`|9|
|`cancelled\_reservations`|2|
|`no\_show\_reservations`|1|
|`total\_room\_nights`|-33|
|`total\_amount\_brl`|5962.41|
|`average\_nightly\_rate\_brl`|0.00|

|`month`|`reservations`|`room\_nights`|`total\_brl`|
|-|-|-|-|
|2025-01|9|-36|5936.90|
|2025-02|1|3|25.51|

|`hotel\_id`|`hotel\_name`|`room\_nights`|`total\_brl`|
|-|-|-|-|
|H01|Hotel Aurora Paulista|3|25.51|
|H02|Hotel Mirante|2|36.90|
|H12|Hotel Miami Beach|1|300.00|
|H10|Hotel Madrid, Gran Vía|1|400.00|
|H09|Hotel Lisboa, Centro|2|660.00|
---
### Resultado esperado:

<!-- AMOSTRA:INICIO -->

|Campo|Valor esperado|
|-|-|
|`total\_reservations`|12|
|`confirmed\_reservations`|9|
|`cancelled\_reservations`|2|
|`no\_show\_reservations`|1|
|`total\_room\_nights`|28|
|`total\_amount\_brl`|7566.00|
|`average\_nightly\_rate\_brl`|270.21|

|`month`|`reservations`|`room\_nights`|`total\_brl`|
|-|-|-|-|
|2025-01|8|25|6684.00|
|2025-02|1|3|882.00|

|`hotel\_id`|`hotel\_name`|`room\_nights`|`total\_brl`|
|-|-|-|-|
|H01|Hotel Aurora Paulista|3|1200.00|
|H02|Hotel Mirante|2|1084.00|
|H12|Hotel Miami Beach|1|1000.00|
|H10|Hotel Madrid, Gran Vía|4|900.00|
|H09|Hotel Lisboa, Centro|3|882.00|

---

### Resumo da Saída dos testes

```powershell
...
FAILED tests/test_currency.py::test_currency_conversion_to_brl - AssertionError: assert '118.45' == '642.00'
FAILED tests/test_nights.py::test_nights_across_month_boundary - assert -28 == 3
FAILED tests/test_regression_report.py::test_regression_report_on_sample - AssertionError: assert {'average_nig...ions': 0, ...} == {'average_nig...ions': 0, ...}
FAILED tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show - AssertionError: assert '120.00' == '100.00'
FAILED tests/test_top_hotels.py::test_top_hotels_are_ranked_descending - AssertionError: assert ['H04', 'H03', 'H02'] == ['H01', 'H02', 'H03']
5 failed, 1 warning in 1.18s
```

---

### 1 - Investigando o primeiro erro do teste `test_currency.py`

```powershell
FAILED tests/test_currency.py::test_currency_conversion_to_brl - AssertionError: assert '118.45' == '642.00'
```

O arquivo de teste `test_currency.py` verifica um faturamento após converter as moedas que não são real para real, então provavelmente o problema está onde essa conversão acontece.  

Verificando o arquivo de conversão `src\services\currency_converter.py` percebo que o calculo está usando divisão:

```python
return Decimal(str(float(amount) / float(rate)))
```

Como estamos convertendo uma moeda o correto é multiplicar o valor pelo câmbio para o real:

```python
return Decimal(str(float(amount) * float(rate)))
```

```powershell
...
FAILED tests/test_nights.py::test_nights_across_month_boundary - assert -28 == 3
FAILED tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show - AssertionError: assert '120.00' == '100.00'
FAILED tests/test_top_hotels.py::test_top_hotels_are_ranked_descending - AssertionError: assert ['H04', 'H03', 'H02'] == ['H01', 'H02', 'H03']
3 failed, 2 passed in 1.03s
```

---

### 2 - Investigando o próximo erro `test_nights`

O teste `test_nights` verifica os dias do faturamento `"A diferença de datas considera a virada de mês, não o dia do mês."` então provavelmente o defeito está onde esse cálculo é feito.  
  
Verificando `src\services\stay_calculator.py` percebo que o calculo só leva em consideração o dia literal de saída e o dia de entrada, que provavelmente causa erros em estadias que vão de um mês para o outro

```python
return (checkout.day - checkin.day) * rooms
```

Provavelmente por isso o test encontra um valor negativo

```powershell
FAILED tests/test_nights.py::test_nights_across_month_boundary - assert -28 == 3
```

O correto seria subtrair as próprias datas já que a classe suporta, e devolver os dias da data bruta

```python
return (checkout - checkin).days * rooms
```

```powershell
...
FAILED tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show - AssertionError: assert '120.00' == '100.00'
FAILED tests/test_top_hotels.py::test_top_hotels_are_ranked_descending - AssertionError: assert ['H04', 'H03', 'H02'] == ['H01', 'H02', 'H03']
2 failed, 3 passed in 1.09s
```

---

### 3 - Investigando `test_status_rules.py`

Observando o arquivo `test_status_rules.py` o erro está acontecendo no quinto assert 
```python
assert report["summary"]["total_amount_brl"] == "100.00"
```
pois o erro detalhado diz
```powershell
_____________________________________________________________________________________________________________________
test_status_rules_exclude_cancelled_and_no_show _____________________________________________________________________________________________________________________
tmp_path = WindowsPath('C:/Users/rafaa/AppData/Local/Temp/pytest-of-rafaa/pytest-3/test_status_rules_exclude_canc0')

    def test_status_rules_exclude_cancelled_and_no_show(tmp_path):
        """Somente CONFIRMED entra nos totais financeiros."""
        report = run_cli(
            tmp_path,
            [
                {"reservation_id": "R-1", "status": "CONFIRMED", "amount": "100.00"},
                {"reservation_id": "R-2", "status": "CANCELLED", "amount": "50.00"},
                {"reservation_id": "R-3", "status": "NO_SHOW", "amount": "20.00"},
            ],
        )
        assert report["summary"]["total_reservations"] == 3
        assert report["summary"]["confirmed_reservations"] == 1
        assert report["summary"]["cancelled_reservations"] == 1
        assert report["summary"]["no_show_reservations"] == 1
>       assert report["summary"]["total_amount_brl"] == "100.00"
E       AssertionError: assert '120.00' == '100.00'
E         
E         - 100.00
E         ?  ^
E         + 120.00
E         ?  ^

tests\test_status_rules.py:20: AssertionError
```

observando os dados do teste

```python
...
[
  {"reservation_id": "R-1", "status": "CONFIRMED", "amount": "100.00"},
  {"reservation_id": "R-2", "status": "CANCELLED", "amount": "50.00"},
  {"reservation_id": "R-3", "status": "NO_SHOW", "amount": "20.00"},
]
...
```

provavelmente NO_SHOW e CONFIRMED estão tendo seus valores somados `AssertionError: assert '120.00' == '100.00'`. Então o erro deve estar onde é verificado o que é faturado ou não.

Verificando `src\services\status_policy.py` está sendo faturado( `is_billable()`) qualquer valor diferente de `CANCELLED`

```python
return status != "CANCELLED"
```

O correto segundo o `README.MD` e o próprio arquivo de teste `test_status_rules.py` é faturar **SOMENTE** os valores `CONFIRMED`, logo:

```python
return status == "CONFIRMED"
```

```powershell
...
FAILED tests/test_top_hotels.py::test_top_hotels_are_ranked_descending - AssertionError: assert ['H04', 'H03', 'H02'] == ['H01', 'H02', 'H03']
1 failed, 4 passed in 1.09s
```

---

### 4 - Investigando `test_top_hotels.py`

Dados usado no teste:

```python
[
  {"reservation_id": "R-1", "hotel_id": "H01", "hotel_name": "Hotel A", "amount": "1000.00"},
  {"reservation_id": "R-2", "hotel_id": "H02", "hotel_name": "Hotel B", "amount": "900.00"},
  {"reservation_id": "R-3", "hotel_id": "H03", "hotel_name": "Hotel C", "amount": "800.00"},
  {"reservation_id": "R-4", "hotel_id": "H04", "hotel_name": "Hotel D", "amount": "700.00"},
]
```

Observando o resultado e os dados, o sistema está entregando a ordem de forma decrescente a partir de amount

```powershell
_____________________________________________________________________________________________________________________ test_top_hotels_are_ranked_descending _____________________________________________________________________________________________________________________
tmp_path = WindowsPath('C:/Users/rafaa/AppData/Local/Temp/pytest-of-rafaa/pytest-4/test_top_hotels_are_ranked_des0')

    def test_top_hotels_are_ranked_descending(tmp_path):
        """O ranking mostra os hotéis de maior valor primeiro."""
        report = run_cli(
            tmp_path,
            [
                {"reservation_id": "R-1", "hotel_id": "H01", "hotel_name": "Hotel A", "amount": "1000.00"},
                {"reservation_id": "R-2", "hotel_id": "H02", "hotel_name": "Hotel B", "amount": "900.00"},
                {"reservation_id": "R-3", "hotel_id": "H03", "hotel_name": "Hotel C", "amount": "800.00"},
                {"reservation_id": "R-4", "hotel_id": "H04", "hotel_name": "Hotel D", "amount": "700.00"},
            ],
            top=3,
        )
>       assert [hotel["hotel_id"] for hotel in report["top_hotels"]] == ["H01", "H02", "H03"]
E       AssertionError: assert ['H04', 'H03', 'H02'] == ['H01', 'H02', 'H03']
E         
E         At index 0 diff: 'H04' != 'H01'
E         Use -v to get more diff

tests\test_top_hotels.py:18: AssertionError
```

Quem entrega isso é o próprio `run_cli()` então a investigação deve seguir para lá, `run_cli` está em `helpers.py` que prepara um teste e roda o próprio `cli.py`, o assert compara ao dado em `report["top_hotels"]` então o problema provavelmente está na construção do report em `src\services\report_builder.py`.

Se o problema está na ordem então o problema deve estar na parte do gerador do relatório que rankeia os hotéis, na linha 91 temos:

```python
ranked_hotels = sorted(hotels.items(), key=lambda item: (item[1]["amount"], item[0]))
top_hotels = [
    {
        "hotel_id": hotel_id,
        "hotel_name": data["hotel_name"],
        "room_nights": data["room_nights"],
        "total_brl": format_money(data["amount"]),
    }
    for hotel_id, data in ranked_hotels[: self._top_n]
]
```

a função sorted por padrão sorteia de forma crescente, mas aceita um parametro para ordenar de forma inversa, então provavelmente o correto seria:

```python
ranked_hotels = sorted(hotels.items(), key=lambda item: (item[1]["amount"], item[0]), reverse=True)
```

testando após a correção:

```powershell
.....
[100%]
5 passed in 6.25s
```

Agora que todos os testes passaram podemos compara o resumo do relatório da amostra após correções.

---

## Resumo do relatório da amostra após correções

|Campo|Valor|
|-|-|
|`total\_reservations`|12|
|`confirmed\_reservations`|9|
|`cancelled\_reservations`|2|
|`no\_show\_reservations`|1|
|`total\_room\_nights`|28|
|`total\_amount\_brl`|7566.00|
|`average\_nightly\_rate\_brl`|270.21|

|`month`|`reservations`|`room\_nights`|`total\_brl`|
|-|-|-|-|
|2025-01|8|25|6684.00|
|2025-02|1|3|882.00|

|`hotel\_id`|`hotel\_name`|`room\_nights`|`total\_brl`|
|-|-|-|-|
|H01|Hotel Aurora Paulista|3|1200.00|
|H02|Hotel Mirante|2|1084.00|
|H12|Hotel Miami Beach|1|1000.00|
|H10|Hotel Madrid, Gran Vía|4|900.00|
|H09|Hotel Lisboa, Centro|3|882.00|

---

### Checklist pós investigação

* \[✅] Ambiente configurado
* \[✅] O CLI roda com Python 3.11 (`$env:PYTHONPATH="src"; python -m onfly_report ...`).
* \[✅] O comando de execução gera `output/report.json` sem erro.
* \[✅] O relatório segue o contrato acima.
* \[✅] `pytest -q` passa com os testes fornecidos.
* \[✅] `entrega/README-ENTREGA.md` preenchido.
* \[✅] Sessão + declaração apenas no caminho com IA; investigação no README-ENTREGA nos dois caminhos.
* \[❌] Repositório público no GitHub com o projeto e o link enviado.