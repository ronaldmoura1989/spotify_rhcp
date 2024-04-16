# Spotify RHCP
Ronald Moura
2024-04-15

## Load Packages

<details>
<summary>Code</summary>

``` r
library(httr)
library(jsonlite)
library(tidyverse)
library(janitor)
library(rstatix)
```

</details>

## Retrieve Token

<details>
<summary>Code</summary>

``` r
#|echo: false

client_id     = "ce56dc9c42a24e6f837301c1e9a943b5"
client_secret = "e8ac01542a504377a367a84795e2bba3"

token_url <- "https://accounts.spotify.com/api/token"
token_response <- POST(
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
<details>
<summary>Code</summary>

``` r
client_id     = "my_id"
client_secret = "my_secret"

token_url <- "https://accounts.spotify.com/api/token"
token_response <- POST(
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

## GET Album’s List

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

#filter only albums
album_response = album_response |> 
  filter(album_group %in% c("album"))
```

</details>

## GET Albums’ Track List

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

track_response = track_response %>%
  select(tracks) %>%
  unnest(tracks, names_repair = "minimal") %>%
  mutate(album_id = str_split(href, "/", simplify = TRUE)[,6]) %>%
  select(album_id, items) %>%
  unnest(items, names_repair = "minimal") %>%
  select(!any_of("linked_from")) %>%
  rename(track_id = id) %>%
  rename(track_name = name)
```

</details>

## GET Track’s Info

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
  assign(i, tibble(get(i)$tracks) %>% unnest_wider(col = 1))
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

## GET Track’s features

<details>
<summary>Code</summary>

``` r
 = ",") #max 100 ids
  
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
  assign(i, tibble(get(i)$audio_features) %>% unnest_wider(col = 1))
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

## Consolidating Dataset

<details>
<summary>Code</summary>

``` r
infos1 = album_response %>%
  select(id, name, total_tracks, release_date, 
         release_date_precision, album_group, external_urls, images) %>%
  unnest(external_urls) %>%
  rename(album_id = id,
         album_name = name,
         album_url = spotify)

infos2 = track_response %>%
  select(album_id, explicit, track_id, track_name, track_number)

infos3 = track_info_response %>%
  rename(track_id = id,
         track_name = name) %>%
  select(track_id, popularity, external_urls) %>%
  unnest_longer(external_urls) %>%
  rename(url_spotify = external_urls) %>%
  select(!any_of("external_urls_id"))

infos4 = track_feature_response %>%
  rename(track_id = id) %>%
  relocate(track_id, .before = danceability)

rhcp = infos1 %>%
  left_join(., infos2, by = "album_id") %>%
  left_join(., infos3, by = "track_id") %>%
  left_join(., infos4, by = "track_id") %>%
  distinct()
```

</details>

## Saving Objects

<details>
<summary>Code</summary>

``` r
save(rhcp, file = "./spotify_rhcp/rhcp.RData")
save.image(file = "./spotify_rhcp/rhcp_ETL.RData")

rm(client_id, client_secret, access_token)
```

</details>
