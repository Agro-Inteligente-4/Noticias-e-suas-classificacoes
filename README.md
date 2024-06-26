# Noticias-e-suas-classificacoes

---
Analise das noticias relacionadas a  cana-de-açúcar
---

Objetivo:

O propósito fundamental deste projeto é realizar uma coleta sistemática de notícias associadas à cana-de-açúcar no site da União da Indústria de Cana-de-Açúcar (UNICA), seguida pela aplicação de técnicas avançadas de mineração de texto. Essas análises detalhadas permitirão a extração de informações valiosas, oferecendo insights cruciais para os agricultores. O foco principal reside em avaliar essas noticias para ajudar os agricultores.

Base de Dados:

A fonte primária de dados para este projeto é o site oficial da UNICA (<https://unica.com.br/>), desenvolvido em colaboração com a Agência Brasileira de Promoção de Exportações e Investimentos (Apex-Brasil). Este site é concebido para ser um centro global de informações sobre produtos relacionados à cana-de-açúcar, destacando seus benefícios em diversas esferas. A parceria entre UNICA e Apex-Brasil visa promover o etanol de cana-de-açúcar brasileiro nas regiões da América do Norte, Europa e Ásia.

A coleta de dados será executada por meio de web scraping, uma técnica robusta e eficaz para extrair informações da web. O uso de robôs de software, como o Selenium, desempenhará um papel essencial na interação eficaz com a página específica de notícias da UNICA (<https://unica.com.br/noticias/>). Essa abordagem garante a precisão na extração dos dados em formato de texto. É imperativo destacar que o web scraping demanda um planejamento cuidadoso, conhecimento técnico e flexibilidade para se adaptar a possíveis alterações nas páginas web de origem.

**Extração de dados sem resumo**

A fim de extrair dados relevantes, foi imprescindível realizar um processo de web scraping nas páginas de notícias no site da UNICA (<https://unica.com.br/noticias/>). Após a conclusão bem-sucedida dessa etapa, os dados coletados foram habilmente armazenados em arquivos CSV, estabelecendo uma base sólida para análises subsequentes.

```{r}
library(rvest)


urls <- c(
#lista de urls das notícias
)


scrape_and_save <- function(url, filename) {
 
  page <- read_html(url)
  
 
  text <- page %>%
    html_nodes(".entry-content p") %>%
    html_text() %>%
    paste(collapse = "\n")
  
  
  write.csv(data.frame(text), file = filename, row.names = FALSE)
  
 
  cat("\nTexto da notícia foi salvo em", filename, "\n")
}


for (i in 1:length(urls)) {
  scrape_and_save(urls[i], paste0("noticia_", i, ".csv"))
}
```

**Analise de sentimento sem o resumo** Após a extração dos dados, o próximo passo crucial foi a criação de um dataframe para organização e estruturação eficiente. A manipulação e pré-processamento dos dados foram etapas subsequentes essenciais, envolvendo a tokenização, remoção de stop words e análise de sentimentos.

Para garantir uma abordagem mais precisa, foi necessário gerar um própria lista de stop words. Isso foi necessário porque o dicionário padrão fornecido pelo R estava removendo palavras relevantes para a análise de sentimento.

A tokenização foi realizada para decompor o texto em unidades significativas, facilitando a análise. A etapa subsequente envolveu a remoção de stop words, com ênfase na preservação de termos pertinentes ao contexto agrícola, cuja lista foi elaborada manualmente para garantir a precisão da análise de sentimento.

A análise de sentimento foi conduzida com base em listas elaboradas manualmente de palavras positivas e negativas relacionadas à agricultura. Essa abordagem personalizada proporcionou uma avaliação mais precisa e adaptada ao cenário específico, levando em consideração as particularidades do setor agrícola.

```{r}
library(tidyverse)
library(tidytext)
library(readtext)
library(SnowballC)
library(stringr)

remover_numeros <- function(texto) {
  texto_sem_numeros <- str_replace_all(texto, "\\b\\d+(,\\d+)?\\b", "")
  texto_sem_numeros <- str_replace_all(texto_sem_numeros, "\\b\\d+º\\b", "") 
  return(texto_sem_numeros)
}

# Função para leitura de arquivos CSV
ler_arquivos_csv <- function(diretorio) {
  arquivos <- list.files(path = diretorio, pattern = ".csv", full.names = TRUE)
  lista_de_dataframes <- lapply(arquivos, function(arquivo) {
    df <- read_csv(arquivo)
    df$doc_id <- basename(arquivo)
    return(df)
  })
  textos <- bind_rows(lista_de_dataframes)
  textos <- textos %>%
    select(doc_id, everything())
  
  return(textos)
}


tokenizar <- function(textos_noticia) {
  textos_noticia %>%
    unnest_tokens(word, text)
}

analise_sentimento <- function(textos_noticia_limpos, palavras_positivas, palavras_negativas) {
  result <- textos_noticia_limpos %>%
    left_join(data_frame(word = c(palavras_positivas, palavras_negativas)), by = "word") %>%
    mutate(sentimento = case_when(
      word %in% palavras_positivas ~ "Positivo",
      word %in% palavras_negativas ~ "Negativo",
      TRUE ~ "Neutro"
    )) %>%
    replace_na(list(sentimento = "Neutro")) %>%
    group_by(doc_id) %>%
    summarize(sentiment = case_when(
      all(is.na(sentimento)) ~ NA_character_,
      TRUE ~ first(sentimento, order_by = desc(!is.na(sentimento)))
    ))
  
  return(result)
}

# Diretório dos arquivos CSV
diretorio <- #diretorio_onde_as_noticias_foram_salvas
# Ler os arquivos CSV
textos_noticia <- ler_arquivos_csv(diretorio)
# Remover números
textos_noticia$text <- sapply(textos_noticia$text, remover_numeros)
# Tokenizar
textos_tokenizados <- tokenizar(textos_noticia)

# Lista de palavras irrelevantes personalizadas em português
stop_words_pt_br_custom <- c(
 "a", "as", "capaz", "sobre", "acima", "de acordo", "através", "na verdade", "depois", "novamente", "contra", "não ", "todos", "permitir", "permite", "quase", "sozinho", "junto", "já", "também", "embora", "sempre", "estou", "entre", "um", "e", "outro", "qualquer", "qualquer um", "de qualquer maneira", "qualquer coisa", "em qualquer lugar", "separado", "aparecer", "apreciar", "apropriado", "está", "não está", "por perto", "como", "de lado", "pergunte", "perguntando", "associado", "em", "disponível", "longe", "terrivelmente", "b", "ser", "tornou-se", "porque", "tornar-se", "torna-se", "foi", "antes", "de antemão", "atrás", "ser", "acreditar", "abaixo", "ao lado", "além de", "melhor", "entre", "além", "ambos", "breve", "mas", "por", "c", "vamos lá", "cs", "veio", "pode", "não pode", "causa", "certo", "certamente", "mudanças", "claramente", "co", "com", "vem", "referente a", "consequentemente", "considerar", "conter", "contém", "correspondente", "poderia", "claro", "atualmente", "d", "definitivamente", "descrito", "apesar de", "fez", "diferente", "faz", "não faz", "fazendo", "não", "feito", "para baixo", "durante", "e", "cada", "edu", "por exemplo", "oito", "qualquer um", "outro", "em outro lugar", "suficiente", "inteiramente", "especialmente", "et", "etc", "mesmo", "sempre", "todos", "tudo", "em todos os lugares", "ex", "exatamente", "exemplo", "exceto", "f", "longe", "poucos", "quinto", "primeiro", "cinco", "seguido", "seguindo", "segue", "para", "anterior", "anteriormente", "adiante", "quatro", "de", "além disso", "g", "obter", "recebe", "recebendo", "dado", "dá", "vai", "foi", "obteve", "saudações", "h", "tinha", "não tinha", "acontece", "dificilmente", "tem", "não tem", "tendo", "ele", "ele é", "olá", "ajuda", "portanto", "ela", "aqui", "daqui em diante", "por meio deste", "aqui", "aqui", "em seguida", "dela", "ela mesma", "oi", "ele", "ele mesmo", "dele", "aqui", "esperançosamente", "como", "entretanto", "no entanto", "eu", "eu", "eu vou", "eu sou", "eu tenho", "ou seja", "se", "ignorado", "imediatamente", "em", "na medida em que", "inc", "de fato", "indicar", "indicado", "indica", "interno", "na medida em que", "em vez disso", "em", "dentro", "é", "não é", "isso", "isso", "isso vai", "é", "é", "em si", "j", "apenas", "k", "manter", "mantém", "mantido", "saber", "conhecido", "sabe", "l", "último", "ultimamente", "mais tarde", "último", "ultimamente", "menos", "para que", "vamos", "como", "gostei", "provavelmente", "pouco", "olhe", "olhando", "olha", "ltd", "m", "principalmente", "muitos", "pode", "talvez", "eu", "quero dizer", "enquanto isso", "apenas", "poderia", "mais", "além disso", "a maioria", "principalmente", "muito", "devo", "meu", "eu mesmo", "n", "nome", "a saber", "nd", "próximo", "quase", "necessário", "necessidade", "precisa", "nem", "nunca", "no entanto", "novo", "próximo", "nove", "não", "ninguém", "não", "nenhum", "ninguém", "nem", "normalmente", "não", "nada", "novo", "agora", "em lugar nenhum", "o", "obviamente", "de", "desligado", "frequentemente", "oh", "ok", "antigo", "ligado", "uma vez", "um", "uns", "apenas", "para", "ou", "outro", "outros", "caso contrário", "deveria", "nosso", "nosso", "nós mesmos", "fora", "fora", "sobre", "no geral", "próprio", "p", "particular", "particularmente", "por", "talvez", "colocado", "por favor", "mais", "possível", "presumivelmente", "provavelmente", "fornece", "q", "que", "bastante", "qv", "r", "em vez disso", "rd", "re", "realmente", "razoavelmente", "em relação a", "independentemente", "respeita", "relativamente", "respectivamente", "certo", "s", "disse", "mesmo", "viu", "digamos", "dizendo", "diz", "em segundo lugar", "ver", "parecer", "parecia", "parecendo", "parece", "visto", "eu mesmo", "eu mesmo", "sensato", "enviado", "sério", "sério", "
sete", "vários", "deve", "ela", "deveria", "não deveria", "já que", "seis", "então", "alguns", "alguém", "de alguma forma", "alguém", "alguma coisa", "em","algum ","momento", "às","vezes", "um pouco", "em algum lugar", "em breve", "desculpe", "especificado", "especifique", "especificando", "ainda", "sub", "tal", "sup", "claro", "t", "ts", "pegue", "tomado", "diga", "tende", "th", "do que", "obrigado", "obrigado", "obrigado", "que", "isso é", "isso é", "o", "deles", "deles", "eles", "eles mesmos", "então", "daí", "lá", "há", "depois disso", "assim", "portanto", "aí", "há", "então", "estes", "eles", "eles", "eles vão", "eles são", "eles têm", "pensam", "terceiro", "isto", "completo", "completamente", "aqueles", "embora", "três", "através", "assim", "para", "juntos", "também", "levou", "em direção a", "tentou", "tenta", "verdadeiramente", "tente", "tentando", "duas vezes", "dois", "u", "un", "sob", "infelizmente", "a menos que", "improvável", "até", "acima", "sobre", "nós", "usar", "usado", "útil", "usa", "usando", "geralmente", "uucp", "v", "valor", "vários", "muito", "via", "viz", "vs", "w", "quer", "quer", "era", "não era", "maneira", "nós", "nós", "nós", "nós", "nós", "nós", "bem-vindos", "bem", "fomos", "fomos", "não fomos", "o que", "o que é", "tanto faz", "quando", "de onde", "quando", "onde", "onde está", "depois de", "enquanto", "por que", "em que", "em que", "onde", "onde", "se", "qual", "enquanto", "enquanto", "para"," onde", "quem", "quem é", "quem", "inteiro", "quem", "de quem", "por que", "irá", "desejará", "desejará", "com", "dentro", "sem", "não irá", "maravilhar-se", "faria", "não faria", "x", "y", "sim", "ainda", "você",  "vai", " é", "tem", "seu", "seu", "mesmo", "z", "zero",
  "de", "a", "ó", "e", "fazer", "pai", "eles", "hum", "pára", "é", "com", "não", "uma", "sistema","operacional",
  "não", "se", "por", "mais", "como", "dos", "como", "mas", "foi", "ao", "ele", "das", "tem", "a", "seu", "sua",
  "ou", "er", "quando", "muito", "há", "não", "já", "está", "UE", "também", "então", "pelo", "pela", "comeu", "isso",
  "ela", "entre", "era", "depois", "sem", "mesmo", "aos", "ter", "seus", "quem", "nas", "meu", "esse", "eles", "estão",
  "você", "tinha", "foram", "essa", "número", "nem", "suas", "meu", "como", "minha", "têm", "numa", "pelos pelos", "elas",
  "havia", "seja", "qual", "será", "nós", "tenho", "o", "deles", "essas", "esses", "pelas", "este", "fosse", "deletar",
  "você", "você", "vocês", "vocês", "eles", "meu", "meus", "teu", "tua", "teus", "tuas", "nosso", "nossa", "nossos",
  "nossos", "dela", "delas", "esta", "estes", "estes", "aquele", "aquela", "aqueles", "aqueles", "isto", "aquilo", "estou","está", "estamos", "estão", "estive", "esteve", "estivemos", "houve", "Houvemos", "houve", "há", "houvéramos", "haja","tenhamos", "hajam", "havia", "houvéssemos", "houve", "haja", "teremos", "hajam", "hajaei", "haverá", "teremos",
  "haveráão", "haveria", "haveríamos", "aconteceriam", "sou", "somos", "são", "era", "éramos", "eram", "fui", "foi",
  "fomos", "foram", "para", "foramos", "seja", "sejamos", "seja", "fosse", "fôssemos", "seria", "para", "formulários",
  "primeiro", "serei", "será", "seremos", "serão", "seria", "seríamos", "seria", "tenho", "tem", "temos", "tem", "tinha",
  "nós", "tinha", "ativo", "teve", "isso", "tive", "tera", "tivéramos", "tenha", "tenhamos", "tenho", "eu tive",
"tivéssemos", "precisam", "tiver", "teremos", "ter", "terei", "terá", "teremos", "terá", "teria", "teríamos", "permanecer","do", "que", "no", "da", "ctc","kg", "em", "na", "um", "os", "as", "ano","até", "à", "sul", "norte", "centro", "início"
)

# Remoção de palavras irrelevantes
textos_noticia_limpos <- textos_tokenizados %>%
  anti_join(data.frame(word = stop_words_pt_br_custom), by = "word")

# Lista de palavras positivas e negativas
palavras_positivas <- c("inovação", "sustentabilidade", "crescimento", "lucro", "eficiência", "desenvolvimento", "qualidade", "progresso", "oportunidade", "investimento", "modernização", "tecnologia", "exportação", "diversificação", "empreendedorismo", "orgulho", "família", "comunidade", "respeito", "responsabilidade", "comprometimento", "confiança", "reconhecimento", "valorização", "motivação", "sucesso", "prosperidade", "rentabilidade", "inovação", "boa", "bom", "parceria", "produtivo", "aprovada", "aprovado", "excelente", "excelência", "sustentável", "aumento", "reduz", "positivo", "positivos", "alta", "comercialização", "venda", "vendas", "superior")
palavras_negativas <- c("pragas", "perda", "prejuízo", "desastre", "estresse", "dificuldades", "adversas", "riscos", "rigorosas", "falha", "concorrência", "intensa", "escassez", "baixos", "baixas", "problemas", "ruim", "declínio", "negativa", "negativo", "queimadas", "nunca", "desvalorização", "endividamento", "inadimplência", "queda", "dificilmente", "infelizmente", "menos", "terrivelmente", "terrivel", "baixo")

data_frame_com_sentimento <- analise_sentimento(textos_noticia_limpos, palavras_positivas, palavras_negativas)
print(data_frame_com_sentimento)
```

A visualização dos resultados anteriores foi aprimorada por meio da criação de um gráfico, proporcionando uma representação mais clara e impactante das análises realizadas.

Para otimizar a compreensão, foi escolhido um formato visual que condensasse as informações de maneira intuitiva. O gráfico destaca os principais insights derivados da análise de sentimento, oferecendo uma perspectiva visual que facilita a interpretação dos dados agrupados.

```{r}
library(ggplot2)

data_frame_com_sentimento$sentiment <- as.numeric(factor(data_frame_com_sentimento$sentiment,
                                                          levels = c("Negativo", "Neutro", "Positivo"),
                                                          labels = c(-1, 0, 1)))

ggplot(data_frame_com_sentimento, aes(x = factor(sentiment), fill = factor(sentiment))) +
  geom_bar() +
  scale_fill_manual(values = c("red", "grey", "green")) +
  labs(title = "Sentiment Distribution", x = "Sentiment", y = "Count") +
  scale_x_discrete(labels = c("Negativo" = -1, "Neutro" = 0, "Positivo" = 1)) +
  theme_minimal()

```

<figure>

  <img src="analise-de-sentimento-sem-resumo.png" alt="analise-de-sentimento-sem-resumo">

<br>


**Modelagem de Tópicos** A modelagem de tópicos é uma técnica poderosa na análise de texto, sendo especialmente relevante no contexto da coleta e análise de notícias relacionadas à cana-de-açúcar. Esta abordagem permite identificar padrões subjacentes nos dados textuais, agrupando automaticamente palavras e documentos relacionados a tópicos específicos. Atravez dessa abordagem conseguimos resultados relevantes como: Identificação de Tópicos Relevantes, Atribuição de Palavras-Chave entre outros.

```{r}
library(ggplot2)
library(dplyr)
library(tidytext)
library(topicmodels)

dtm <- textos_noticia_limpos %>%
  count(doc_id, word) %>%
  cast_dtm(document = doc_id, term = word, value = n)

# Create an LDA model with k=2
my_lda <- LDA(dtm, k = 3, control = list(seed = 1234))

# Extract probabilities by topic per word (beta)
topics <- tidy(my_lda, matrix = "beta")

# Assuming 'ap_topics' is your data frame with topic probabilities (beta)

ap_top_terms <- topics %>%
  group_by(topic) %>%
  slice_max(beta, n = 5) %>% 
  ungroup() %>%
  arrange(topic, -beta)

# Create a plot
ap_top_terms %>%
  mutate(term = reorder_within(term, beta, topic)) %>%
  ggplot(aes(beta, term, fill = factor(topic))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~ topic, scales = "free") +
  scale_y_reordered() +
  labs(title = "Principais termos para cada tópico", x = "Beta Probabilidade", y = "Termos")
```
<figure>

  <img src="modelagem-de-topicos-sem-resumo.png" alt="modelagem-de-topicos-sem-resumo">

<br>


*Mudando a abordagem na tentativa de conseguir melhores resultados* **Extração e Resumo Automático dos Dados**

Aprimorando as abordagens anteriores, observamos que os resultados não alcançaram a relevância desejada. Contudo, uma virada positiva foi alcançada ao incorporar a geração de resumos no momento da extração de dados. Mantendo a técnica de web scraping para a coleta, a inovação reside no processo de resumir cada notícia antes de armazená-la nos arquivos.

Essa abordagem mais refinada não apenas otimizou a eficiência da análise, mas também proporcionou uma visão mais concentrada e informativa de cada notícia. A técnica de resumo desempenhou um papel fundamental na destilação das informações essenciais, tornando os resultados mais significativos e relevantes para a análise subsequente.

A consideração de resumos no processo de extração não apenas agilizou o tratamento dos dados, mas também contribuiu para mitigar desafios associados à análise de grandes volumes de texto. Esta inovação demonstra uma abordagem adaptativa e refinada para lidar com as complexidades inerentes à mineração de dados, resultando em insights mais precisos e de maior valor para os agricultores e demais interessados no contexto da cana-de-açúcar.

```{r}
library(rvest)
library(stringr)
library(lexRankr)
library(dplyr)

urls <- c(
#lista de urls das notícias
)

scrape_and_process <- function(url) {
  page <- read_html(url)
  
  text <- page %>%
    html_nodes(".entry-content p") %>%
    html_text() %>%
    paste(collapse = "\n")
  
  chapters <- stringi::stri_split_boundaries(text, type = "sentence")
  
  chapters_clean <- stringr::str_remove_all(chapters, "[A-Z]{2,} {0,1}[0-9]{0,}")
  
  top3s <- lexRankr::lexRank(chapters_clean,
                             n = 3,
                             continuous = TRUE) %>%
    dplyr::pull(sentence) %>%
    stringr::str_remove_all("[^[:alnum:] ]") %>%
    stringr::str_squish()
  
  return(top3s)
}

for (i in 1:length(urls)) {
  result <- scrape_and_process(urls[i])
  write.csv(data.frame(result), file = paste0("resumo", 63+ i, ".csv"), row.names = FALSE)
  cat("\nTop 3 sentences of", urls[i], "have been saved in", paste0("resumo", i, ".csv"), "\n")
}
```

**Analise de sentimento com resumo** Após a extração dos dados e gerar o resumos, o próximo passo crucial foi a criação de um dataframe para organização e estruturação eficiente. A manipulação e pré-processamento dos dados foram etapas subsequentes essenciais, envolvendo a tokenização, remoção de stop words e análise de sentimentos.

Para garantir uma abordagem mais precisa, foi necessário gerar um própria lista de stop words. Isso foi necessário porque o dicionário padrão fornecido pelo R estava removendo palavras relevantes para a análise de sentimento.

A tokenização foi realizada para decompor o texto em unidades significativas, facilitando a análise. A etapa subsequente envolveu a remoção de stop words, com ênfase na preservação de termos pertinentes ao contexto agrícola, cuja lista foi elaborada manualmente para garantir a precisão da análise de sentimento.

A análise de sentimento foi conduzida com base em listas elaboradas manualmente de palavras positivas e negativas relacionadas à agricultura. Essa abordagem personalizada proporcionou uma avaliação mais precisa e adaptada ao cenário específico, levando em consideração as particularidades do setor agrícola.

```{r}
library(tidyverse)
library(tidytext)
library(readtext)
library(SnowballC)
library(stringr)

remover_numeros <- function(texto) {
  texto_sem_numeros <- str_replace_all(texto, "\\b\\d+(,\\d+)?\\b", "")
  texto_sem_numeros <- str_replace_all(texto_sem_numeros, "\\b\\d+º\\b", "") 
  return(texto_sem_numeros)
}

# Função para leitura de arquivos CSV
ler_arquivos_csv <- function(diretorio) {
  arquivos <- list.files(path = diretorio, pattern = ".csv", full.names = TRUE)
  lista_de_dataframes <- lapply(arquivos, function(arquivo) {
    df <- read_csv(arquivo)
    df$doc_id <- basename(arquivo)
    return(df)
  })
  textos <- bind_rows(lista_de_dataframes)
  textos <- textos %>%
    select(doc_id, everything())
  
  return(textos)
}


tokenizar <- function(textos_resumo) {
  textos_resumo %>%
    unnest_tokens(word, text)
}

analise_sentimento <- function(textos_resumo_limpos, palavras_positivas, palavras_negativas) {
  print("Dimensions of textos_resumo_limpos:")
  print(dim(textos_resumo_limpos))
  
  print("Dimensions of palavras_positivas:")
  print(dim(data.frame(word = palavras_positivas)))
  
  print("Dimensions of palavras_negativas:")
  print(dim(data.frame(word = palavras_negativas)))

  result <- textos_resumo_limpos %>%
    left_join(data_frame(word = c(palavras_positivas, palavras_negativas)), by = "word") %>%
    mutate(sentimento = case_when(
      word %in% palavras_positivas ~ "Positivo",
      word %in% palavras_negativas ~ "Negativo",
      TRUE ~ "Neutro"
    )) %>%
    replace_na(list(sentimento = "Neutro")) %>%
    group_by(doc_id) %>%
    summarize(sentiment = case_when(
      all(is.na(sentimento)) ~ NA_character_,
      TRUE ~ first(sentimento, order_by = desc(!is.na(sentimento)))
    ))
  
  return(result)
}


# Diretório dos arquivos CSV
diretorio <- #diretorio_onde_as_noticias_foram_salvas

# Ler os arquivos CSV
textos_resumo <- ler_arquivos_csv(diretorio)
# Remover números
textos_resumo$text <- sapply(textos_resumo$result, remover_numeros)
# Tokenizar
textos_tokenizados_resumo <- tokenizar(textos_resumo)

# Lista de palavras irrelevantes personalizadas em português
stop_words_pt_br_custom <- c(
 "a", "as", "capaz", "sobre", "acima", "de acordo", "através", "na verdade", "depois", "novamente", "contra", "não ", "todos", "permitir", "permite", "quase", "sozinho", "junto", "já", "também", "embora", "sempre", "estou", "entre", "um", "e", "outro", "qualquer", "qualquer um", "de qualquer maneira", "qualquer coisa", "em qualquer lugar", "separado", "aparecer", "apreciar", "apropriado", "está", "não está", "por perto", "como", "de lado", "pergunte", "perguntando", "associado", "em", "disponível", "longe", "terrivelmente", "b", "ser", "tornou-se", "porque", "tornar-se", "torna-se", "foi", "antes", "de antemão", "atrás", "ser", "acreditar", "abaixo", "ao lado", "além de", "melhor", "entre", "além", "ambos", "breve", "mas", "por", "c", "vamos lá", "cs", "veio", "pode", "não pode", "causa", "certo", "certamente", "mudanças", "claramente", "co", "com", "vem", "referente a", "consequentemente", "considerar", "conter", "contém", "correspondente", "poderia", "claro", "atualmente", "d", "definitivamente", "descrito", "apesar de", "fez", "diferente", "faz", "não faz", "fazendo", "não", "feito", "para baixo", "durante", "e", "cada", "edu", "por exemplo", "oito", "qualquer um", "outro", "em outro lugar", "suficiente", "inteiramente", "especialmente", "et", "etc", "mesmo", "sempre", "todos", "tudo", "em todos os lugares", "ex", "exatamente", "exemplo", "exceto", "f", "longe", "poucos", "quinto", "primeiro", "cinco", "seguido", "seguindo", "segue", "para", "anterior", "anteriormente", "adiante", "quatro", "de", "além disso", "g", "obter", "recebe", "recebendo", "dado", "dá", "vai", "foi", "obteve", "saudações", "h", "tinha", "não tinha", "acontece", "tem", "não tem", "tendo", "ele", "ele é", "olá", "ajuda", "portanto", "ela", "aqui", "daqui em diante", "por meio deste", "aqui", "aqui", "em seguida", "dela", "ela mesma", "oi", "ele", "ele mesmo", "dele", "aqui", "esperançosamente", "como", "entretanto", "no entanto", "eu", "eu", "eu vou", "eu sou", "eu tenho", "ou seja", "se", "ignorado", "imediatamente", "em", "na medida em que", "inc", "de fato", "indicar", "indicado", "indica", "interno", "na medida em que", "em vez disso", "em", "dentro", "é", "não é", "isso", "isso", "isso vai", "é", "é", "em si", "j", "apenas", "k", "manter", "mantém", "mantido", "saber", "conhecido", "sabe", "l", "último", "ultimamente", "mais tarde", "último", "ultimamente", "menos", "para que", "vamos", "como", "gostei", "provavelmente", "pouco", "olhe", "olhando", "olha", "ltd", "m", "principalmente", "muitos", "pode", "talvez", "eu", "quero dizer", "enquanto isso", "apenas", "poderia", "mais", "além disso", "a maioria", "principalmente", "muito", "devo", "meu", "eu mesmo", "n", "nome", "a saber", "nd", "próximo", "quase", "necessário", "necessidade", "precisa", "nem", "nunca", "no entanto", "novo", "próximo", "nove", "não", "ninguém", "não", "nenhum", "ninguém", "nem", "normalmente", "não", "nada", "novo", "agora", "em lugar nenhum", "o", "obviamente", "de", "desligado", "frequentemente", "oh", "ok", "antigo", "ligado", "uma vez", "um", "uns", "apenas", "para", "ou", "outro", "outros", "caso contrário", "deveria", "nosso", "nosso", "nós mesmos", "fora", "fora", "sobre", "no geral", "próprio", "p", "particular", "particularmente", "por", "talvez", "colocado", "por favor", "mais", "possível", "presumivelmente", "provavelmente", "fornece", "q", "que", "bastante", "qv", "r", "em vez disso", "rd", "re", "realmente", "razoavelmente", "em relação a", "independentemente", "respeita", "relativamente", "respectivamente", "certo", "s", "disse", "mesmo", "viu", "digamos", "dizendo", "diz", "em segundo lugar", "ver", "parecer", "parecia", "parecendo", "parece", "visto", "eu mesmo", "eu mesmo", "sensato", "enviado", "sério", "sério", "
sete", "vários", "deve", "ela", "deveria", "não deveria", "já que", "seis", "então", "alguns", "alguém", "de alguma forma", "alguém", "alguma coisa", "em","algum ","momento", "às","vezes", "um pouco", "em algum lugar", "em breve", "desculpe", "especificado", "especifique", "especificando", "ainda", "sub", "tal", "sup", "claro", "t", "ts", "pegue", "tomado", "diga", "tende", "th", "do que", "obrigado", "obrigado", "obrigado", "que", "isso é", "isso é", "o", "deles", "deles", "eles", "eles mesmos", "então", "daí", "lá", "há", "depois disso", "assim", "portanto", "aí", "há", "então", "estes", "eles", "eles", "eles vão", "eles são", "eles têm", "pensam", "terceiro", "isto", "completo", "completamente", "aqueles", "embora", "três", "através", "assim", "para", "juntos", "também", "levou", "em direção a", "tentou", "tenta", "verdadeiramente", "tente", "tentando", "duas vezes", "dois", "u", "un", "sob", "infelizmente", "a menos que", "improvável", "até", "acima", "sobre", "nós", "usar", "usado", "útil", "usa", "usando", "geralmente", "uucp", "v", "valor", "vários", "muito", "via", "viz", "vs", "w", "quer", "quer", "era", "não era", "maneira", "nós", "nós", "nós", "nós", "nós", "nós", "bem-vindos", "bem", "fomos", "fomos", "não fomos", "o que", "o que é", "tanto faz", "quando", "de onde", "quando", "onde", "onde está", "depois de", "enquanto", "por que", "em que", "em que", "onde", "onde", "se", "qual", "enquanto", "enquanto", "para"," onde", "quem", "quem é", "quem", "inteiro", "quem", "de quem", "por que", "irá", "desejará", "desejará", "com", "dentro", "sem", "não irá", "maravilhar-se", "faria", "não faria", "x", "y", "sim", "ainda", "você",  "vai", " é", "tem", "seu", "seu", "mesmo", "z", "zero",
  "de", "a", "ó", "e", "fazer", "pai", "eles", "hum", "pára", "é", "com", "não", "uma", "sistema","operacional",
  "não", "se", "por", "mais", "como", "dos", "como", "mas", "foi", "ao", "ele", "das", "tem", "a", "seu", "sua",
  "ou", "er", "quando", "muito", "há", "não", "já", "está", "UE", "também", "então", "pelo", "pela", "comeu", "isso",
  "ela", "entre", "era", "depois", "sem", "mesmo", "aos", "ter", "seus", "quem", "nas", "meu", "esse", "eles", "estão",
  "você", "tinha", "foram", "essa", "número", "nem", "suas", "meu", "como", "minha", "têm", "numa", "pelos pelos", "elas",
  "havia", "seja", "qual", "será", "nós", "tenho", "o", "deles", "essas", "esses", "pelas", "este", "fosse", "deletar",
  "você", "você", "vocês", "vocês", "eles",  "meus", "teu", "tua", "teus", "tuas", "nosso", "nossa", "nossos",
  "nossos", "dela", "delas", "esta", "estes", "estes", "aquele", "aquela", "aqueles", "aqueles", "isto", "aquilo", "estou","está", "estamos", "estão", "estive", "esteve", "estivemos", "houve", "Houvemos", "houve", "há", "houvéramos", "haja","tenhamos", "hajam", "havia", "houvéssemos", "houve", "haja", "teremos", "hajam", "hajaei", "haverá", "teremos",
  "haveráão", "haveria", "haveríamos", "aconteceriam", "sou", "somos", "são", "era", "éramos", "eram", "fui", "foi",
  "fomos", "foram", "para", "foramos", "seja", "sejamos", "seja", "fosse", "fôssemos", "seria", "para", "formulários",
  "primeiro", "serei", "será", "seremos", "serão", "seria", "seríamos", "seria", "tenho", "tem", "temos", "tem", "tinha",
  "nós", "tinha", "ativo", "teve", "isso", "tive", "tera", "tivéramos", "tenha", "tenhamos", "tenho", "eu tive",
"tivéssemos", "precisam", "tiver", "teremos", "ter", "terei", "terá", "teremos", "terá", "teria", "teríamos", "permanecer","do", "que", "no", "da", "ctc","kg", "em", "na", "um", "os", "as", "ano","até", "à", "sul", "norte", "centro", "início"
)

# Remoção de palavras irrelevantes
textos_resumo_limpos <- textos_tokenizados_resumo %>%
  anti_join(data.frame(word = stop_words_pt_br_custom), by = "word")

# Lista de palavras positivas e negativas
palavras_positivas <- c("inovação", "sustentabilidade", "crescimento", "lucro", "eficiência", "desenvolvimento", "qualidade", "progresso", "oportunidade", "investimento", "modernização", "tecnologia", "exportação", "diversificação", "empreendedorismo", "orgulho", "família", "comunidade", "respeito", "responsabilidade", "comprometimento", "confiança", "reconhecimento", "valorização", "motivação", "sucesso", "prosperidade", "rentabilidade", "boa", "bom", "parceria", "produtivo", "aprovada", "aprovado", "excelente", "excelência", "sustentável", "aumento", "reduz", "positivo", "positivos", "alta", "comercialização", "venda", "vendas", "superior", "colaboração", "incentivo", "colheita", "abundância", "resiliência", "eficiência energética", "estar", "agronegócio", "modernidade", "inclusão", "fertilidade", "inovação agrícola", "bem","sucedido", "fomento", "auto-sustentável", "biodiversidade", "boas práticas", "renovação", "prosperidade", "rural", "conquistas", "agricultura", "familiar", "rendimento", "solidez", "riqueza","natural", "valor","agregado", "empoderamento", "satisfação", "gratidão", "segurança","alimentar", "valor","sustentável", "vendas", "mercado","externo"
)
palavras_negativas <- c("pragas", "perda", "prejuízo", "desastre", "estresse", "dificuldades", "adversas", "riscos", "rigorosas", "falha", "concorrência", "intensa", "escassez", "baixos", "baixas", "problemas", "ruim", "declínio", "negativa", "negativo", "queimadas", "nunca", "desvalorização", "endividamento", "inadimplência", "queda", "dificilmente", "infelizmente", "menos", "terrivelmente", "terrivel", "baixo", "dificilmente", "desastre","ambiental", "epidemia", "contaminação", "poluição", "erosão", "solo", "deslizamento", "peste", "praga","insetos", "seca", "inundações", "incêndio","florestal", "falta","recursos", "fome", "crise", "hídrica", "tempestade","destrutiva", "insustentáveis", "desmatamento", "malsucedida", "tóxicos", "mudanças","climáticas", "condições","adversas", "política","desfavorável", "altos", "custos", "preços","baixos","mercado", "comercialização","desfavorável", "condições", "extremas", "impactos", "desapropriação", "aumento","custos", "resistência", "desvalorização","cambial", "excesso","regulamentação", "insegurança", "falta", "crédito", "conflitos", "declínio", "subsídios","reduzidos", "concorrência","desleal")

data_frame_com_sentimento_resumo <- analise_sentimento(textos_resumo_limpos, palavras_positivas, palavras_negativas)
print(data_frame_com_sentimento_resumo)
```

A visualização dos resultados anteriores foi aprimorada por meio da criação de um gráfico, proporcionando uma representação mais clara e impactante das análises realizadas.

Para otimizar a compreensão, foi escolhido um formato visual que condensasse as informações de maneira intuitiva. O gráfico destaca os principais insights derivados da análise de sentimento, oferecendo uma perspectiva visual que facilita a interpretação dos dados agrupados.

Ao perceber a diferença nos resultados, fica evidente que a inclusão dos resumos aprimorou significativamente a capacidade de definir com maior precisão o sentimento de cada notícia. Essa observação destaca a importância crucial do processo de resumir no refinamento da análise, proporcionando uma compreensão mais nítida e contextualizada das emoções transmitidas por cada peça de informação.

A habilidade aprimorada de capturar o tom e o conteúdo essencial de cada notícia através dos resumos não apenas elevou a qualidade da análise de sentimento, mas também demonstrou uma abordagem estratégica para lidar com grandes volumes de dados textuais. Esse refinamento não só otimiza a eficiência do processo, mas também contribui para insights mais robustos e relevantes, melhorando significativamente a capacidade de oferecer informações valiosas aos agricultores no cenário da cana-de-açúcar.

```{r}
library(ggplot2)

data_frame_com_sentimento_resumo$sentiment <- as.numeric(factor(data_frame_com_sentimento_resumo$sentiment,
                                                          levels = c("Negativo", "Neutro", "Positivo"),
                                                          labels = c(-1, 0, 1)))

ggplot(data_frame_com_sentimento_resumo, aes(x = factor(sentiment), fill = factor(sentiment))) +
  geom_bar() +
  scale_fill_manual(values = c("red", "grey", "green")) +
  labs(title = "Sentiment Distribution", x = "Sentiment", y = "Count") +
  scale_x_discrete(labels = c("Negativo" = -1, "Neutro" = 0, "Positivo" = 1)) +
  theme_minimal()
```
<figure>

  <img src="analise-de-sentimento.png" alt="analise-de-sentimento">

<br>

**Modelagem de Tópicos** A modelagem de tópicos é uma técnica poderosa na análise de texto, sendo especialmente relevante no contexto da coleta e análise de notícias relacionadas à cana-de-açúcar. Esta abordagem permite identificar padrões subjacentes nos dados textuais, agrupando automaticamente palavras e documentos relacionados a tópicos específicos. Atravez dessa abordagem conseguimos resultados relevantes como: Identificação de Tópicos Relevantes, Atribuição de Palavras-Chave entre outros.

```{r}

library(ggplot2)
library(dplyr)
library(tidytext)
library(topicmodels)

dtm <- textos_resumo_limpos %>%
  count(doc_id, word) %>%
  cast_dtm(document = doc_id, term = word, value = n)

# Create an LDA model with k=2
my_lda <- LDA(dtm, k = 5, control = list(seed = 1234))

# Extract probabilities by topic per word (beta)
topics <- tidy(my_lda, matrix = "beta")

# Assuming 'ap_topics' is your data frame with topic probabilities (beta)

ap_top_terms <- topics %>%
  group_by(topic) %>%
  slice_max(beta, n = 5) %>% 
  ungroup() %>%
  arrange(topic, -beta)

# Create a plot
ap_top_terms %>%
  mutate(term = reorder_within(term, beta, topic)) %>%
  ggplot(aes(beta, term, fill = factor(topic))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~ topic, scales = "free") +
  scale_y_reordered() +
  labs(title = "Principais termos para cada tópico", x = "Beta Probabilidade", y = "Termos")

# Print the topics
#print(topics)
```
<figure>

  <img src="modelagem-de-topicos.png" alt="modelagem-de-topicos">

<br>
