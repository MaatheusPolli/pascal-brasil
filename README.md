# PascalBrasil

[![FPC](https://github.com/<user>/pascal-brasil/actions/workflows/fpc.yml/badge.svg)](https://github.com/<user>/pascal-brasil/actions/workflows/fpc.yml)
[![Delphi](https://github.com/<user>/pascal-brasil/actions/workflows/delphi.yml/badge.svg)](https://github.com/<user>/pascal-brasil/actions/workflows/delphi.yml)
![LicenÃ§a MIT](https://img.shields.io/badge/licen%C3%A7a-MIT-blue)

Biblioteca Delphi open-source com utilitÃ¡rios brasileiros: validaÃ§Ã£o de documentos (CPF, CNPJ **alfanumÃ©rico**, IE das 27 UFs), mÃ¡scaras, payload **PIX copia e cola** e consulta de **CEP** e **CNPJ** â€” com testes automatizados, sem dependÃªncias alÃ©m da RTL, e compatÃ­vel com Free Pascal.

## InstalaÃ§Ã£o

```
boss install github.com/<user>/pascal-brasil
```

Sem Boss: adicione `src` e suas subpastas ao *Library Path*. Detalhes em [docs/instalacao.md](docs/instalacao.md).

## MÃ³dulos

| MÃ³dulo | Unidades | Rede? |
|--------|----------|:-----:|
| Documentos | `PascalBrasil.Cpf`, `PascalBrasil.Cnpj`, `PascalBrasil.InscricaoEstadual`, `PascalBrasil.Modulo11` | nÃ£o |
| MÃ¡scaras | `PascalBrasil.Mask` | nÃ£o |
| PIX | `PascalBrasil.Pix.Payload`, `PascalBrasil.Pix.Tlv`, `PascalBrasil.Pix.Crc16` | nÃ£o |
| Consultas | `PascalBrasil.Cep`, `PascalBrasil.Cnpj.Consulta`, `PascalBrasil.Http` | sim |
| Base | `PascalBrasil.Types`, `PascalBrasil.Exceptions` | nÃ£o |

As unidades sÃ£o independentes: usar sÃ³ documentos nÃ£o linka nada de rede.

## Exemplos

### Documentos

```pascal
uses PascalBrasil.Cpf, PascalBrasil.Cnpj, PascalBrasil.InscricaoEstadual;

TCpf.IsValid('529.982.247-25');            // True
TCpf.Format('52998224725');                // '529.982.247-25'
TCpf.Generate(True);                       // CPF vÃ¡lido aleatÃ³rio, formatado

TCnpj.IsValid('11.222.333/0001-81');       // True
TCnpj.IsValid('12.ABC.345/01DE-35');       // True â€” CNPJ alfanumÃ©rico (2026)

TInscricaoEstadual.IsValid('SP', '110.042.490.114'); // True
TCpf.Validate('123.456.789-00');           // levanta EDocumentoInvalido
```

### MÃ¡scaras

```pascal
uses PascalBrasil.Mask;

TMask.Apply('###.###.###-##', '52998224725'); // '529.982.247-25'
TMask.Apply('###.###.###-##', '52998');       // '529.98' (parcial, bom para OnChange)
TMask.Telefone('11912345678');                // '(11) 91234-5678'
TMask.Cep('01310100');                        // '01310-100'
TMask.Moeda(1234.56);                         // 'R$ 1.234,56' (independe do locale)
```

### PIX copia e cola

```pascal
uses PascalBrasil.Pix.Payload;

var
  LOpcoes: TPixOptions;
begin
  LOpcoes := Default(TPixOptions);
  LOpcoes.Chave := '123e4567-e12b-12d1-a456-426655440000';
  LOpcoes.NomeRecebedor := 'Fulano de Tal';
  LOpcoes.Cidade := 'BrasÃ­lia';   // acentos removidos automaticamente
  LOpcoes.Valor := 10.50;         // 0 = pagador digita o valor
  LOpcoes.Txid := 'PEDIDO42';     // vazio = '***'
  Memo1.Text := TPix.GerarPayload(LOpcoes);
end;
```

### Consultas de CEP e CNPJ

```pascal
uses PascalBrasil.Cep, PascalBrasil.Exceptions;

var
  LClient: TCepClient;
  LEndereco: TEndereco;
begin
  LClient := TCepClient.Create;   // ViaCEP, com fallback para BrasilAPI
  try
    try
      LEndereco := LClient.Consultar('01310-100');
      ShowMessage(LEndereco.Logradouro + ' - ' + LEndereco.Cidade);
    except
      on E: ETimeout do ShowMessage('ServiÃ§o lento, tente de novo');
      on E: EConsultaFalhou do
        if E.StatusCode = 404 then ShowMessage('CEP nÃ£o encontrado')
        else ShowMessage(E.Message);
    end;
  finally
    LClient.Free;
  end;
end;
```

`TCnpjClient` funciona igual (BrasilAPI â†’ ReceitaWS). Timeout, tentativas e ordem dos providers sÃ£o configurÃ¡veis por `Config` e `Providers`; o cliente HTTP Ã© injetÃ¡vel (`IHttpClient`).

### ExceÃ§Ãµes

| Classe | Quando |
|--------|--------|
| `EDocumentoInvalido` | documento com formato ou dÃ­gito invÃ¡lido (`Validate`, `Format`, consultas) |
| `EParametroInvalido` | entrada invÃ¡lida (ex.: campo obrigatÃ³rio do PIX vazio) |
| `EConsultaFalhou` | falha de consulta; `StatusCode = 404` quando nÃ£o encontrado |
| `ETimeout` | consulta excedeu o tempo (descende de `EConsultaFalhou`) |

Todas descendem de `EPascalBrasil`. ReferÃªncia completa em [docs/api.md](docs/api.md).

## Testes

- **Delphi** (DUnitX): `powershell -File build.ps1` â€” requer `dcc32` de linha de comando (nÃ£o disponÃ­vel na Community Edition; nela, abra `tests/PascalBrasilTests.dpr` na IDE e rode).
- **Free Pascal**: `bash run-tests-fpc.sh` â€” executa a mesma suÃ­te via um shim mÃ­nimo de DUnitX (`tests/fpc`).

Os testes sÃ£o mÃ©todos `published` das fixtures, descobertos tanto pelo DUnitX quanto pelo shim do FPC (que nÃ£o suporta atributos). As consultas sÃ£o testadas com `Mocks.HttpClient` e fixtures JSON â€” a suÃ­te roda offline.

Vetores de referÃªncia: CRC16 `"123456789"` â†’ `29B1`; payload PIX idÃªntico ao exemplo do Manual do BR Code (Bacen); CNPJ alfanumÃ©rico `12.ABC.345/01DE-35` da Receita; IE com os exemplos do SINTEGRA para as 27 UFs.

## Samples

- `samples/ConsoleDemo` â€” todos os mÃ³dulos em console (Delphi e FPC). `--online` consulta CEP/CNPJ de verdade.
- `samples/VclDemo` â€” formulÃ¡rio VCL: validar documento, consultar CEP, gerar PIX.

## Compatibilidade

Delphi 10.4+ e Free Pascal 3.2+ (`{$MODE DELPHI}`). No FPC, `string` Ã© tratada como UTF-8 (convenÃ§Ã£o Lazarus) e as consultas HTTPS usam `fphttpclient` + OpenSSL (no Windows, as DLLs do OpenSSL precisam estar no PATH).

## LicenÃ§a

[MIT](LICENSE)


## Licença

### Uso Pessoal
Este projeto é licenciado sob a **Licença MIT** para uso pessoal, educacional e não comercial.

### Uso Comercial
Se você deseja usar este projeto comercialmente, deve obter uma licença comercial do autor.

**Uso comercial inclui:**
- Incorporar em um produto comercial
- Oferecer como serviço
- Usar em atividades com fins lucrativos
- Implantação empresarial

**Para obter uma licença comercial:**
Email: ma.polli@hotmail.com
