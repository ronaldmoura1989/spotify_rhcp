# Spotify RHCP
Ronald Moura
2024-04-15

## Carregamento de pacotes

Para este exercício, usaremos as bilbiotecas abaixo:

<details>
<summary>Code</summary>

``` r
library(httr)      #realizar consultas a bases de dados através APIs
library(jsonlite)  #converter arquivos JSON em data frames
library(tidyverse) #manipulação dos dados
library(janitor)   #utilitário para auxiliar na manipulação de dados
library(rstatix)   #análises estatísticas
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

Uma vez que você tem em mãos a ID e o secret, basta seguir com a geração
do token diretamente pelo R:

<details>
<summary>Code</summary>

``` r
client_id     = "my_id"
client_secret = "my_secret"

token_url = "https://accounts.spotify.com/api/token"
token_response = POST(
  url = token_url,
  body = list(
    grant_type = "client_credentials",
    client_id = client_id,
    client_secret = client_secret
  ),
  encode = "form"
)

access_token = content(token_response)$access_token
```

</details>

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
endpoint  = "https://api.spotify.com/v1/artists"
album_url = paste0(endpoint, "/", artist_id, "/albums?market=US&limit=50") #max 50 albums

album_response = GET(
  url = album_url,
  add_headers(Authorization = paste("Bearer", access_token))
)

album_response = fromJSON(content(album_response, as = "text"))
album_response = tibble(album_response$items)

#filter only original studio albums
album_response = album_response |> 
  filter(album_group %in% c("album"))
```

</details>

## Acessar a lista das músicas

A partir do id dos álbuns obtidos através da consulta anterior, agora
podemos passá-los como argumento para realizar uma nova busca. Desta
vez, nosso objetivo é conseguir a lista das músicas (e suas ids) para
cada álbum. Para esta consulta, existe um limite de 20 álbuns por vez.
Deste modo, implementei uma iteração, que irá realizar a consulta para
cada 20 álbuns listados, e agregar todos os resultados em um único
objeto **track_response**:

<details>
<summary>Code</summary>

``` r
if(length(album_response$id) >= 20){ 

  start = c(seq(from = 1,
                 to = length(album_response$id),
                 by = 20))
  
  end = c(seq(from = 20,
               to = length(album_response$id),
               by = 20),
           length(album_response$id))
  
  parts = paste0("part", seq(1, length(start)))
  
  batches1 = data.frame(parts,
                        start,
                        end)
} else {
  start = 1
  end = length(album_response$id)
  parts = 1
  
  batches1 = data.frame(parts,
                        start,
                        end)
}

for(i in 1:nrow(batches1)) {

  album_ids = paste0(album_response$id[batches1$start[i]:batches1$end[i]], 
                     collapse = "%2C") #max 20 ids
  endpoint  = "https://api.spotify.com/v1/albums"
  tracks_url = paste0(endpoint, "?ids=", album_ids, "&market=US")
  
  assign(paste0("tracks_response",i), 
         GET(url = tracks_url,
             add_headers(Authorization = paste("Bearer", access_token))
            )
  )

}

for(i in ls(pattern = "tracks_response")) {
  
  assign(i, fromJSON(content(get(i), as = "text")))
  assign(i, tibble(get(i)$albums))
  
}

#put all parts together
track_response = 
  eval(parse(text = 
               paste0("bind_rows(", paste0(ls(pattern = "tracks_response"), 
                                           collapse = ","), ")")))

rm(list = ls(pattern = "tracks_response"))

track_response = track_response |>
  select(tracks) |>
  unnest(tracks, names_repair = "minimal") |>
  mutate(album_id = str_split(href, "/", simplify = TRUE)[,6]) |>
  select(album_id, items) |>
  unnest(items, names_repair = "minimal") |>
  select(!any_of("linked_from")) |>
  rename(track_id = id) |>
  rename(track_name = name)
```

</details>

## Acessar informação básicas sobre as músicas

Com os ids de cada uma das músicas listados, conseguimos obter uma busca
para algumas informações básicas sobre cada canção. Aqui, na
documentação, diz que a busca pode ser feita utilizando até 100 ids de
músicas por vez, porém comigo só funcionou com até 50 ids. Deste modo, a
implementação do passo anterior para uma iteração e depois agregação de
cada parte em um único objeto foi reutilizada:

<details>
<summary>Code</summary>

``` r
start = c(seq(from = 1, 
    to = length(track_response$track_id), 
    by = 50))

end = c(seq(from = 50, 
            to = length(track_response$track_id), 
            by = 50), 
        length(track_response$track_id))

parts = paste0("part", seq(1, length(start)))

batches2 = data.frame(parts,
                     start,
                     end)

for(i in 1:nrow(batches2)) {
  track_ids = paste0(track_response$track_id[batches2$start[i]:batches2$end[i]],
                     collapse = ",") #max 100 ids but worked only with 50
  
  endpoint  = "https://api.spotify.com/v1/tracks?"
  tracks_info = paste0(endpoint, "&market=US", "&ids=", track_ids)
  
  assign(paste0("tracks_info_response", i), 
         GET(url = tracks_info,
             add_headers(Authorization = paste("Bearer", access_token))
             )
  )
  
  #tracks_info_response = content(tracks_info_response)
  assign(paste0("tracks_info_response", i),
         content(get(paste0("tracks_info_response", i)))
  )
  
}

for(i in ls(pattern = "tracks_info_response")){
  assign(i, tibble(get(i)$tracks) |> unnest_wider(col = 1))
}
rm(i)

#put all parts together
track_info_response = 
  eval(parse(text = 
               paste0("bind_rows(", paste0(ls(pattern = "tracks_info_response"), 
                                           collapse = ","), ")")))

rm(list = ls(pattern = "tracks_info_response"))
```

</details>

## Acessar informações detalhadas sobre as músicas

A última consulta à base de dados do spotify foi para obter informações
mais detalhadas sobre cada faixa, de cada álbum. Agora sim, nesta busca
foi possível passar 100 ids por consulta, e mais uma vez a iteração foi
necessária:

<details>
<summary>Code</summary>

``` r
start = c(seq(from = 1, 
              to = length(track_response$track_id), 
              by = 100))

end = c(seq(from = 100, 
            to = length(track_response$track_id), 
            by = 100), 
        length(track_response$track_id))

parts = paste0("part", seq(1, length(start)))

batches3 = data.frame(parts,
                      start,
                      end)


for(i in 1:nrow(batches3)) {
  track_ids = paste0(track_response$track_id[batches3$start[i]:batches3$end[i]],
                     collapse = ",") #max 100 ids
  
  endpoint  = "https://api.spotify.com/v1/audio-features?"
  tracks_feature = paste0(endpoint, "ids=", track_ids)
  
  assign(paste0("tracks_feature_response", i), 
         GET(url = tracks_feature,
             add_headers(Authorization = paste("Bearer", access_token))
         )
  )
  
  #tracks_feature_response = content(tracks_feature_response)
  assign(paste0("tracks_feature_response", i),
         content(get(paste0("tracks_feature_response", i)))
  )
  
}

for(i in ls(pattern = "tracks_feature_response")){
  assign(i, tibble(get(i)$audio_features) |> unnest_wider(col = 1))
}
rm(i)

#put all parts together
track_feature_response = 
  eval(parse(text = 
               paste0("bind_rows(", paste0(ls(pattern = "tracks_feature_response"), 
                                           collapse = ","), ")")))

rm(list = ls(pattern = "tracks_feature_response"))
```

</details>

## Consolidando o dataset

Após as quatro consultas, conseguimos reunir algums dados de cada uma
delas em um objeto consolidado com as informações mais relevantes para
análisar a discografia do RHCP, segundo o Spotify:

<details>
<summary>Code</summary>

``` r
infos1 = album_response |>
  select(id, name, total_tracks, release_date, 
         release_date_precision, album_group, external_urls, images) |>
  unnest(external_urls) |>
  rename(album_id = id,
         album_name = name,
         album_url = spotify)

infos2 = track_response |>
  select(album_id, explicit, track_id, track_name, track_number)

infos3 = track_info_response |>
  rename(track_id = id,
         track_name = name) |>
  select(track_id, popularity, external_urls) |>
  unnest_longer(external_urls) |>
  rename(url_spotify = external_urls) |>
  select(!any_of("external_urls_id"))

infos4 = track_feature_response |>
  rename(track_id = id) |>
  relocate(track_id, .before = danceability)

rhcp = infos1 |>
  left_join(., infos2, by = "album_id") |>
  left_join(., infos3, by = "track_id") |>
  left_join(., infos4, by = "track_id") |>
  distinct()
```

</details>

## Salvando objetos

Só para manter um backup, e não ter que repetir todas as buscas
desnecessariamente, costumo salvar o .RData com todos os objetos criados
e um outro só com o dataset.

<details>
<summary>Code</summary>

``` r
rm(client_id, client_secret, access_token)

save(rhcp, file = "rhcp.RData")
save.image(file = "rhcp_ETL.RData")
```

</details>
