# PascalBrasil

[![FPC](https://github.com/<user>/pascal-brasil/actions/workflows/fpc.yml/badge.svg)](https://github.com/<user>/pascal-brasil/actions/workflows/fpc.yml)
[![Delphi](https://github.com/<user>/pascal-brasil/actions/workflows/delphi.yml/badge.svg)](https://github.com/<user>/pascal-brasil/actions/workflows/delphi.yml)
[![Licença: MIT pessoal + comercial](https://img.shields.io/badge/licen%C3%A7a-MIT%20pessoal%20%2B%20comercial-blue.svg)](LICENSE)

Biblioteca Delphi com utilitários brasileiros: validação de documentos (CPF, CNPJ **alfanumérico**, IE das 27 UFs), máscaras, payload **PIX copia e cola** e consulta de **CEP** e **CNPJ** — com testes automatizados, sem dependências além da RTL, e compatível com Free Pascal.

## Instalação

```
boss install github.com/<user>/pascal-brasil
```

Sem Boss: adicione `src` e suas subpastas ao *Library Path*. Detalhes em [docs/instalacao.md](docs/instalacao.md).

## Módulos

| Módulo | Unidades | Rede? |
|--------|----------|:-----:|
| Documentos | `PascalBrasil.Cpf`, `PascalBrasil.Cnpj`, `PascalBrasil.InscricaoEstadual`, `PascalBrasil.Modulo11` | não |
| Máscaras | `PascalBrasil.Mask` | não |
| PIX | `PascalBrasil.Pix.Payload`, `PascalBrasil.Pix.Tlv`, `PascalBrasil.Pix.Crc16` | não |
| Consultas | `PascalBrasil.Cep`, `PascalBrasil.Cnpj.Consulta`, `PascalBrasil.Http` | sim |
| Base | `PascalBrasil.Types`, `PascalBrasil.Exceptions` | não |

As unidades são independentes: usar só documentos não linka nada de rede.

## Exemplos

### Documentos

```pascal
uses PascalBrasil.Cpf, PascalBrasil.Cnpj, PascalBrasil.InscricaoEstadual;

TCpf.IsValid('529.982.247-25');            // True
TCpf.Format('52998224725');                // '529.982.247-25'
TCpf.Generate(True);                       // CPF válido aleatório, formatado

TCnpj.IsValid('11.222.333/0001-81');       // True
TCnpj.IsValid('12.ABC.345/01DE-35');       // True — CNPJ alfanumérico (2026)

TInscricaoEstadual.IsValid('SP', '110.042.490.114'); // True
TCpf.Validate('123.456.789-00');           // levanta EDocumentoInvalido
```

### Máscaras

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
  LOpcoes.Cidade := 'Brasília';   // acentos removidos automaticamente
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
      on E: ETimeout do ShowMessage('Serviço lento, tente de novo');
      on E: EConsultaFalhou do
        if E.StatusCode = 404 then ShowMessage('CEP não encontrado')
        else ShowMessage(E.Message);
    end;
  finally
    LClient.Free;
  end;
end;
```

`TCnpjClient` funciona igual (BrasilAPI → ReceitaWS). Timeout, tentativas e ordem dos providers são configuráveis por `Config` e `Providers`; o cliente HTTP é injetável (`IHttpClient`).

### Exceções

| Classe | Quando |
|--------|--------|
| `EDocumentoInvalido` | documento com formato ou dígito inválido (`Validate`, `Format`, consultas) |
| `EParametroInvalido` | entrada inválida (ex.: campo obrigatório do PIX vazio) |
| `EConsultaFalhou` | falha de consulta; `StatusCode = 404` quando não encontrado |
| `ETimeout` | consulta excedeu o tempo (descende de `EConsultaFalhou`) |

Todas descendem de `EPascalBrasil`. Referência completa em [docs/api.md](docs/api.md).

## Testes

- **Delphi** (DUnitX): `powershell -File build.ps1` — requer `dcc32` de linha de comando (não disponível na Community Edition; nela, abra `tests/PascalBrasilTests.dpr` na IDE e rode).
- **Free Pascal**: `bash run-tests-fpc.sh` — executa a mesma suíte via um shim mínimo de DUnitX (`tests/fpc`).

Os testes são métodos `published` das fixtures, descobertos tanto pelo DUnitX quanto pelo shim do FPC (que não suporta atributos). As consultas são testadas com `Mocks.HttpClient` e fixtures JSON — a suíte roda offline.

Vetores de referência: CRC16 `"123456789"` → `29B1`; payload PIX idêntico ao exemplo do Manual do BR Code (Bacen); CNPJ alfanumérico `12.ABC.345/01DE-35` da Receita; IE com os exemplos do SINTEGRA para as 27 UFs.

## Samples

- `samples/ConsoleDemo` — todos os módulos em console (Delphi e FPC). `--online` consulta CEP/CNPJ de verdade.
- `samples/VclDemo` — formulário VCL: validar documento, consultar CEP, gerar PIX.

## Compatibilidade

Delphi 10.4+ e Free Pascal 3.2+ (`{$MODE DELPHI}`). No FPC, `string` é tratada como UTF-8 (convenção Lazarus) e as consultas HTTPS usam `fphttpclient` + OpenSSL (no Windows, as DLLs do OpenSSL precisam estar no PATH).

## Status

Etapas 1 a 6 concluídas (documentos, IE das 27 UFs, máscaras, PIX, consultas CEP/CNPJ e samples).

Pendências conhecidas:

- publicar a release

## Licença

Este projeto usa **licença dupla**. O texto completo está em [LICENSE](LICENSE).

- **Uso pessoal, educacional e não comercial:** livre, nos termos da licença MIT (mantenha o aviso de copyright).
- **Uso comercial:** exige licença comercial. Isso inclui usar em produto ou serviço comercial, vender acesso ou derivados, usar em atividades com fins lucrativos e implantar em empresas.

Contato para licença comercial: ma.polli@hotmail.com
