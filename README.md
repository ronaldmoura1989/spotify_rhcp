# Spotify RHCP
Ronald Moura
2024-04-15

## Carregamento de pacotes

Para este exercício, usaremos as bilbiotecas abaixo:

<details>
<summary>Code</summary>

``` r
library(httr)      #realizar consultas a bases de dados através APIs
library(tidyverse) #manipulação dos dados
```

</details>

## Geração de token para acesso à API

O primeiro passo é adicionar a função de desenvolvedor na sua conta do
spotify, através deste
[link](https://developer.spotify.com/ "https://developer.spotify.com/"),
onde será enviado uma mensagem de confirmação para o seu e-mail
cadastrado. Comigo aconteceu desta confirmação chegar, mas dar erro.
Ficou dois dias assim, mas depois de algumas tentativas, deu certo.

Ao validar sua conta para desenvolvedor, você terá acesso a um
*dashboard* onde será possível registrar um app, através do botão
**create app**. Basta preencher o formulário com algumas informações e a
aplicação estará registrada. Isto irá fazer com que seja gerado uma
**ID** e um **secret**. Essas duas informações são necessárias para
gerar um token de acesso à API. Aqui, é importante copiar estas
informações e não compartilha-las.

Uma vez que você tem em mãos a ID e o secret, é possível criar um na
pasta de trabalho chamado *.venv*, onde salvamos essas informações de
modo que não fiquem escritas explicitamente no código. O arquivo fica
com assim:

> CLIENT_ID=\<ID\>
>
> CLIENT_SECRET=\<SECRET\>

Onde é só substituir \<ID\> e \<SECRET\> pelos valores da sua conta. não
precisa colocar entre aspas, apenas os valores. A seguir, podemos
prosseguir com a geração do token diretamente pelo R:

<details>
<summary>Code</summary>

``` r
readRenviron(".venv") 

token_url = "https://accounts.spotify.com/api/token"
token_res = request(token_url) |> 
  req_auth_basic(username = Sys.getenv("CLIENT_ID"), 
                 password = Sys.getenv("CLIENT_SECRET")) |> 
  req_body_form(grant_type = "client_credentials") |> 
  req_perform()

access_token = token_res |> 
  resp_body_json() |> 
  pluck("access_token") 
```

</details>

O objeto *access_token* irá armazenar o token gerado, que vale por 60
minutos.

## Acessar a lista dos álbuns

Nesta etapa, realizaremos a primeira consulta, que é para obter a lista
de álbuns do RHCP. Neste endpoint, é possível consultar até 50 álbuns de
uma vez, a partir do id do artista, que é possível encontrar na URL da
página do próprio artista. No caso do Red Hot, a URL da página do
Spotify é <https://open.spotify.com/artist/0L8ExT028jH3ddEcZwqJJ5>,
consequentemente a ID é ‘0L8ExT028jH3ddEcZwqJJ5’. É esta sequencia que
passamos como valor da variável **artist_id** no código abaixo:

<details>
<summary>Code</summary>

``` r
artist_id = "0L8ExT028jH3ddEcZwqJJ5"
album_url  = "https://api.spotify.com/v1/artists"
album_res = request(album_url) |> 
  req_url_path_append(artist_id) |> 
  req_url_path_append("albums") |> 
  req_url_query(market = "US",               # albuns lançados no mercado dos EUA
                limit  = 50,                 # limite de resultados  
                include_groups = "album") |> # incluir apenas álbuns de estúdio
  req_auth_bearer_token(access_token) |>     # autorização via token
  req_perform()

album_info = album_res |> 
  resp_body_json() |> 
  pluck("items") |> 
  map_dfr(
    \(x) {
      tibble(id   = x |> pluck("id"),
             name = x |> pluck("name")
      )
    }
  )
```

</details>

O resultado dessa requisição será um dataframe com o id e o nome de cada
álbum do RHCP. Note que, na requisição, coloquei somente para receber os
resultados referentes aos “álbuns de estúdio”. Portanto, coletâneas e
singles não serão requisitados. O limite máximo por requisição que a API
impõe é de 50 álbuns. Outra opção interessante é filtrar pelo local onde
o álbum foi lançado. Pode acontecer de haver um álbum exclusivo de um
determinado mercado. No caso do RHCP, eu sei que todos os álbuns de
estúdio sairam pelo menos nos Estados Unidos. Mais detalhes neste
[link](https://developer.spotify.com/documentation/web-api/reference/get-an-artists-albums).

## Acessar a lista das músicas

A partir do id dos álbuns obtidos através da consulta anterior, agora
podemos passá-los como argumento para realizar uma nova busca. Desta
vez, nosso objetivo é conseguir a lista das músicas (e suas ids) para
cada álbum. Para esta consulta, existe um limite de 20 álbuns por vez.
Como o Red Hot tem 17 discos de estúdio registrados até agora, não vamos
ter problemas com esse limite.

<details>
<summary>Code</summary>

``` r
tracks_url  = "https://api.spotify.com/v1/albums"
tracks_res  = request(tracks_url) |> 
  req_url_query(ids = paste0(album_info$id, collapse = ",")) |>  #max 20 ids
  req_url_query(market = "US") |>
  req_auth_bearer_token(access_token) |> 
  req_perform()

track_list = tracks_res |> 
  resp_body_json() |> 
  pluck(1) |> 
  map_dfr(
    \(x) {
      tibble(album_external_urls     = x |> pluck("external_urls") |> list(),
             album_id                = x |> pluck("id"),
             album_images            = x |> pluck("images")        |> list(),
             album_name              = x |> pluck("name"),
             album_popularity        = x |> pluck("popularity"),
             album_release_date      = x |> pluck("release_date"),
             tracks                  = x |> pluck("tracks", "items"),
             album_type              = x |> pluck("type"),
             album_uri               = x |> pluck("uri")
      )
    }
  ) |> 
  unnest_wider(tracks, names_repair = "minimal") |> 
  rename_with(~ paste0("track_",.x), !starts_with("album_"))
```

</details>

O objeto final chamado de *track_list* contém mais algumas informações
sobre os álbuns, e as informações sobre as faixas.

## Acessar informação básicas sobre as músicas

Com os ids de cada uma das músicas listados, conseguimos obter uma busca
para algumas informações básicas sobre cada canção. No total, temos 287
músicas registradas, e na documentação, diz que a busca pode ser feita
utilizando até 100 ids de músicas por vez, porém comigo só funcionou com
até 50 ids quando testei direto na versão web da API. Deste modo, para a
implementação ser mais simples, utilizei o mode de requisição em
paralelo, através da função *req_perform_parallel()*:

<details>
<summary>Code</summary>

``` r
track_info_url  = "https://api.spotify.com/v1/tracks"
track_info_res = map(track_list$id, \(x){
    request(track_info_url) |> 
      req_url_query(ids = x) |> #max 100 ids
      req_url_query(market = "US") |>
      req_auth_bearer_token(access_token) |> 
      req_perform()
  }) |>  
  req_perform_parallel()
  
track_info = map_df(track_info_res, resp_body_json)|> 
  unnest_wider(tracks, names_repair = "minimal") |> 
  select(id, popularity) |> 
  rename_with(~ paste0("track_",.x), everything())
```

</details>

A função *req_perform_parallel* executa várias requisições ao mesmo
tempo, a partir de uma lista contendo as requisições, e gerando no final
um objeto também do tipo lista com todas as respostas. Vale frisar que
esse método funcionou bem neste caso, porém pode ser que cause problemas
de muitas requisições simultâneas para a API. No final, foi criado o
objeto *track_info*, com o id das músicas, que já tinha, mais uma
variável chamada **popularity**, uma medidada de zero a 100 sobre o
quanto a canção é popular. Mais detalhes neste [link](#0).

## Acessar informações detalhadas sobre as músicas

A última consulta à base de dados do spotify foi para obter informações
mais detalhadas sobre cada faixa, de cada álbum. E mais uma vez a busca
em paralelo foi utilizada:

<details>
<summary>Code</summary>

``` r
track_detail_url  = "https://api.spotify.com/v1/audio-features"
track_detail_res = map(track_list$id, \(x){
  request(track_detail_url) |> 
    req_url_query(ids = x) |>  #max 100 ids
    req_url_query(market = "US") |>
    req_auth_bearer_token(access_token) |> 
    req_perform()
}) |>  
  req_perform_parallel()

track_detail = map_df(track_detail_res, resp_body_json) |> 
  unnest_wider(audio_features, names_repair = "minimal") |> 
  rename(href = track_href) |> 
  rename_with(~ paste0("track_",.x), everything()) |> 
  select(-c(track_duration_ms, 
            track_href, track_type, track_uri))
```

</details>

O objeto final contem diversas informações e métricas sobre as faixas,
como duração, se tem conteúdo explícito, tonalidade, tempo, etc. Mais
detalhes
[aqui](https://developer.spotify.com/documentation/web-api/reference/get-several-audio-features).

## Consolidando o dataset

Após as quatro consultas, conseguimos reunir algums dados de cada uma
delas em um objeto consolidado com as informações mais relevantes para
análisar a discografia do RHCP, segundo o Spotify:

<details>
<summary>Code</summary>

``` r
rhcp = track_list %>% 
  left_join(., track_info, by = "track_id") %>%
  left_join(., track_detail, by = "track_id") %>%
  relocate(c(album_id, album_name), .before = album_external_urls)
```

</details>

## Salvando o objeto com o dataset

Só para manter um backup, e não ter que repetir todas as buscas
desnecessariamente.

<details>
<summary>Code</summary>

``` r
save(rhcp, file = "rhcp.RData")  
```

</details>

## Considerações finais

A combinação do *tidyverse* com o pacote *httr2* faz com que um processo
de ETL (*extract, transform and load*) seja bastante eficaz,
especialmente a utilização de funções do pacote *purrr* para manipulação
das respostas em JSON.
