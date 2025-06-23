# Patrimônio histórico de Juiz de Fora

Desenvolvemos um projeto em Python para mapear os bens tombados do patrimônio histórico de Juiz de Fora, Minas Gerais. 
Utilizamos técnicas de web scraping, processamento de dados e visualização geográfica, para criar um mapa interativo que 
facilita a exploração dos marcos culturais do município.

### Importamos as bibliotecas

``` python
import numpy as np
import pandas as pd
import osmnx as ox
import requests
from bs4 import BeautifulSoup
import os
from groq import Groq
from PyPDF2 import PdfReader
import tempfile
import base64
import re
from thefuzz import fuzz, process
import folium
from folium.plugins import Search, MarkerCluster 
import json

from dotenv import load_dotenv
```

### Baixamos a lista

Extraímos os dados da Wikipedia, onde existe uma lista completa dos bens tombados na cidade.
Utilizamos a biblioteca *BeautifulSoup* para analisar o HTML e extrair as informações da tabela.
Ao mesmo tempo, convertemos as coordenadas no formato DMS (graus, minutos, segundos) para decimal.

``` python
def extrair_bens_tombados():
    url = "https://pt.wikipedia.org/wiki/Lista_de_bens_tombados_em_Juiz_de_Fora" 
    response = requests.get(url)
    soup = BeautifulSoup(response.text, 'html.parser')

    table = soup.find('table', {'class': 'wikitable'})
    bens_list = []

    for row in table.find_all('tr')[1:]:
        cols = row.find_all(['td', 'th'])
        if len(cols) >= 5:
            nome_bem = cols[1].get_text(strip=True)
            coord_texto = cols[3].get_text(strip=True)

            try:
                lat, lon = extrair_coordenadas(coord_texto)
                if lat is not None and lon is not None:
                    bens_list.append({
                        'Bem': nome_bem,
                        'Latitude': lat,
                        'Longitude': lon
                    })
            except Exception as e:
                print(f"Erro ao processar linha: {e}")
                continue

    return bens_list

def extrair_coordenadas(texto):
    if not texto.strip():
        return None, None

    try:
        partes = texto.split(',')
        if len(partes) < 2:
            return None, None

        lat_str = partes[0].strip()
        lon_str = partes[1].strip()

        def dms_para_decimal(dms):
            dms = dms.replace('°', ' ').replace('′', ' ').replace('″', '').strip()
            graus, minutos, segundos = map(float, dms.split()[:3])
            direcao = dms.split()[-1]

            decimal = abs(graus) + minutos / 60 + segundos / 3600
            if direcao in ['S', 'O']:
                decimal *= -1

            return decimal

        lat = dms_para_decimal(lat_str)
        lon = dms_para_decimal(lon_str)

        return lat, lon
    except Exception as e:
        print(f"Erro ao processar coordenada: {texto} - {e}")
        return None, None

# Extração dos dados
bens = extrair_bens_tombados()
```

### Convertemos a lista em dataframe

``` python
df = pd.DataFrame(bens)
df.to_csv("bens_tombados_jf.csv", index=False, sep=";", encoding="utf-8")
```

### Inspecionamos o dataframe

``` python
print(df.head())
```

``` python
                                                     Bem   Latitude  Longitude
    0  Acervo documental do "Fundo Câmara Municipal d... -21.755278 -43.344167
    1                                   Agência Bradesco -21.761111 -43.348056
    2                                   Agência Bradesco -21.761111 -43.348056
    3                                     Alfândega Seca -21.761389 -43.343056
    4                  Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
```

``` python
df.shape
```

    (140, 3)

``` python
print(df.isnull().sum())
```

    Bem          0
    Latitude     0
    Longitude    0
    dtype: int64

``` python
df.isna().sum()
```

    Bem          0
    Latitude     0
    Longitude    0
    dtype: int64

### Limpeza de dados

**Removemos as linhas com valores em branco, nulos ou faltantes:**

``` python
df = df.replace(['', np.nan], np.nan).dropna()
print(df.head())
```

``` python
                                                     Bem   Latitude  Longitude
    0  Acervo documental do "Fundo Câmara Municipal d... -21.755278 -43.344167
    1                                   Agência Bradesco -21.761111 -43.348056
    2                                   Agência Bradesco -21.761111 -43.348056
    3                                     Alfândega Seca -21.761389 -43.343056
    4                  Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
```

**Removemos as linhas duplicadas:**

``` python
df = df.drop_duplicates()
print(df.head())
```

``` python
                                                     Bem   Latitude  Longitude
    0  Acervo documental do "Fundo Câmara Municipal d... -21.755278 -43.344167
    1                                   Agência Bradesco -21.761111 -43.348056
    3                                     Alfândega Seca -21.761389 -43.343056
    4                  Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
    5    Antiga Estação Ferroviária da Central do Brasil -22.903611 -43.191111
```

**Removemos a primeiro registro, pois não se trata de um bem imóvel:**

``` python
df = df.iloc[1:]
print(df.head())
```

``` python
                                                   Bem   Latitude  Longitude
    1                                 Agência Bradesco -21.761111 -43.348056
    3                                   Alfândega Seca -21.761389 -43.343056
    4                Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
    5  Antiga Estação Ferroviária da Central do Brasil -22.903611 -43.191111
    6      Antiga Estação Ferroviária de Santos Dumont -21.455833 -43.549722
```

**Criamos o dataframe contendo os bens rotulados:**

``` python
df_temp = df[~df['Bem'].str.contains('Imóvel', case=False)]
print(df_temp.head())
```

``` python
                                                   Bem   Latitude  Longitude
    1                                 Agência Bradesco -21.761111 -43.348056
    3                                   Alfândega Seca -21.761389 -43.343056
    4                Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
    5  Antiga Estação Ferroviária da Central do Brasil -22.903611 -43.191111
    6      Antiga Estação Ferroviária de Santos Dumont -21.455833 -43.549722
```

### Filtramos os bens com denominação genérica

``` python
df_imovel = df[df['Bem'].str.contains('Imóvel', case=False)]
print(df_imovel.head())
```

``` python
                                                  Bem   Latitude  Longitude
    60  Imóvel à Avenida Barão do Rio Branco, nº 3029 -21.768333 -43.347222
    61  Imóvel à Avenida Barão do Rio Branco, nº 3146 -21.769722 -43.347778
    62  Imóvel à Avenida Barão do Rio Branco, nº 3263 -21.770556 -43.346944
    63  Imóvel à Avenida Barão do Rio Branco, nº 3408 -21.771944 -43.347222
    64               Imóvel à Avenida Brasil, nº 2001 -21.758889 -43.343889
```

### Utilizamos modelo de linguagem para manipular arquivo pdf

``` python
# Carregamos a chave API da Groq
load_dotenv()

# Configuração inicial
GROQ_API_KEY = os.getenv("GROQ_API_KEY")  # Substitua pela sua chave da Groq
MODEL_NAME = "meta-llama/llama-4-maverick-17b-128e-instruct"  # Ou "gemini-1.5-pro" se disponível
PDF_PATH = "bens_tombados_17092021.pdf"

# Função para extrair texto do PDF
def extract_text_from_pdf(pdf_path):
    text = ""
    with open(pdf_path, "rb") as f:
        reader = PdfReader(f)
        for page in reader.pages:
            text += page.extract_text() + "\n"
    return text

# Função para processar o texto com o LLM
def process_with_groq(text_content):
    client = Groq(api_key=GROQ_API_KEY)
    
    prompt = f"""
    Você é um assistente especializado em extrair dados estruturados de documentos. 
    Extraia uma tabela com os bens tombados do seguinte texto, com três colunas:
    1. id (número sequencial)
    2. endereço (se disponível)
    3. nome_edificio (descrição do bem tombado)

    Formato esperado:
    id | endereço | nome_edificio
    ---|----------|-------------
    1  | Rua X, 123 | Edifício ABC

    Texto para análise:
    {text_content}
    """

    response = client.chat.completions.create(
        messages=[{"role": "user", "content": prompt}],
        model=MODEL_NAME,
        temperature=0.3
    )
    
    return response.choices[0].message.content

# Função para converter a resposta em DataFrame
def parse_response_to_df(response_text):
    lines = [line.split('|') for line in response_text.strip().split('\n') if '|' in line]
    
    # Remover cabeçalho se existir
    if 'id' in lines[0][0].lower():
        lines = lines[1:]
    
    data = []
    for line in lines:
        # Limpar cada campo
        cleaned = [item.strip() for item in line]
        
        # Garantir que temos 3 colunas
        if len(cleaned) == 3:
            data.append({
                'id': cleaned[0],
                'endereco': cleaned[1],
                'nome_edificio': cleaned[2]
            })
    
    return pd.DataFrame(data)

# Fluxo principal
def main():
    # Extrair texto do PDF
    print("Extraindo texto do PDF...")
    text_content = extract_text_from_pdf(PDF_PATH)
    
    # Processar com Groq
    print("Processando com o modelo LLM...")
    response = process_with_groq(text_content)
    
    # Converter para DataFrame
    print("Convertendo para DataFrame...")
    df_funalfa = parse_response_to_df(response)
    
    # Salvar como CSV
    output_path = "bens_tombados.csv"
    df_funalfa.to_csv(output_path, index=False, encoding='utf-8-sig')
    print(f"Dados salvos em {output_path}")
    
    # Mostrar preview
    print("\nPreview dos dados:")
    print(df_funalfa.head())

if __name__ == "__main__":
    main()
```

::: {.output .stream .stdout}
    Extraindo texto do PDF...
    Processando com o modelo LLM...
    Convertendo para DataFrame...
    Dados salvos em bens_tombados.csv

    Preview dos dados:
        id                                     endereco  \
    0  ---                                   ----------   
    1    1                             Rua Halfeld, s/n   
    2    2                   Praça Dr. João Pessoa, s/n   
    3    3                           Parque e Acervo do   
    4    4  Rua Espírito Santo, 467 e Usina de Marmelos   

                                           nome_edificio  
    0                                      -------------  
    1   Edifício do antigo Fórum, atual Câmara Municipal  
    2                               Cine Theatro Central  
    3                             Museu Mariano Procópio  
    4  Remanescentes das antigas instalações da Cia. ...  
:::
:::

::: {#568fe67f-76ec-4aad-b269-df7c9bd1a880 .cell .markdown}
### Extraímos uma amostra do data frame
:::

::: {#3b9f6323-5ae1-4a35-b3c0-8d8583eb5092 .cell .code execution_count="39"}
``` python
df_tombados = pd.read_csv("bens_tombados.csv")
print(df_tombados.head())
```

::: {.output .stream .stdout}
        id                                     endereco  \
    0  ---                                   ----------   
    1    1                             Rua Halfeld, s/n   
    2    2                   Praça Dr. João Pessoa, s/n   
    3    3                           Parque e Acervo do   
    4    4  Rua Espírito Santo, 467 e Usina de Marmelos   

                                           nome_edificio  
    0                                      -------------  
    1   Edifício do antigo Fórum, atual Câmara Municipal  
    2                               Cine Theatro Central  
    3                             Museu Mariano Procópio  
    4  Remanescentes das antigas instalações da Cia. ...  
:::
:::

::: {#cff9bfdd-0fc6-4695-bada-67c6100e5012 .cell .markdown}
### Removemos os registros em branco
:::

::: {#7ef86d1b-56fd-4a37-b41d-112022d57115 .cell .code execution_count="40"}
``` python
df_tombados_filtrado = df_tombados.dropna()
print(df_tombados_filtrado.head())
```

::: {.output .stream .stdout}
        id                                     endereco  \
    0  ---                                   ----------   
    1    1                             Rua Halfeld, s/n   
    2    2                   Praça Dr. João Pessoa, s/n   
    3    3                           Parque e Acervo do   
    4    4  Rua Espírito Santo, 467 e Usina de Marmelos   

                                           nome_edificio  
    0                                      -------------  
    1   Edifício do antigo Fórum, atual Câmara Municipal  
    2                               Cine Theatro Central  
    3                             Museu Mariano Procópio  
    4  Remanescentes das antigas instalações da Cia. ...  
:::
:::

::: {#b9b50a14-5eb5-44a0-a5dc-97153cefed55 .cell .markdown}
### Atribuímos nomes específicos aos registros genéricos
:::

::: {#eb581d87-cbb5-4022-a0d9-e2c5833bacd8 .cell .code execution_count="41"}
``` python
# Configuração do fuzzy matching
LIMITE_SIMILARIDADE = 85  # Ajuste conforme necessidade (0-100)

def padronizar_endereco(endereco):
    # Remover termos irrelevantes e padronizar
    endereco = re.sub(r'(Imóvel à |nº ?|,|\.)', '', str(endereco))
    endereco = re.sub(r'\s+', ' ', endereco).strip().lower()
    
    # Padronizar variações comuns
    substituicoes = {
        'av ': 'avenida ',
        'av. ': 'avenida ',
        'r ': 'rua ',
        'pc ': 'praça ',
        'estr ': 'estrada '
    }
    for orig, subst in substituicoes.items():
        endereco = endereco.replace(orig, subst)
    
    return endereco

def encontrar_melhor_match(endereco_alvo, opcoes):
    # Padronizar o endereço alvo
    endereco_alvo_padrao = padronizar_endereco(endereco_alvo)
    
    # Padronizar todas as opções de referência
    opcoes_padrao = [padronizar_endereco(op) for op in opcoes]
    
    # Usar fuzzywuzzy para encontrar a melhor correspondência
    melhor_match, score = process.extractOne(
        endereco_alvo_padrao,
        opcoes_padrao,
        scorer=fuzz.token_set_ratio
    )
    
    # Retornar o endereço original (não padronizado) se atender ao limite
    if score >= LIMITE_SIMILARIDADE:
        return opcoes[opcoes_padrao.index(melhor_match)]
    return None

def atualizar_com_fuzzy_matching(df, df_imovel):
    # Criar cópia explícita para evitar SettingWithCopyWarning
    df_imovel_clean = df_imovel.copy()
    
    # Criar lista de endereços de referência
    enderecos_referencia = df['endereco'].dropna().unique()
    
    # Criar coluna temporária para os matches
    df_imovel_clean['match_encontrado'] = None
    
    # Para cada endereço em df_imovel, encontrar o melhor match
    for idx, row in df_imovel_clean.iterrows():
        endereco_imovel = row['Bem']
        melhor_match = encontrar_melhor_match(endereco_imovel, enderecos_referencia)
        
        if melhor_match:
            # Encontrar o nome do edifício correspondente
            nome_edificio = df.loc[df['endereco'] == melhor_match, 'nome_edificio'].values[0]
            df_imovel_clean.at[idx, 'match_encontrado'] = nome_edificio
    
    # Atualizar a coluna 'Bem' onde encontramos matches
    df_imovel_clean['Bem'] = df_imovel_clean.apply(
        lambda x: x['match_encontrado'] if pd.notna(x['match_encontrado']) else x['Bem'],
        axis=1
    )
    
    # Remover coluna temporária
    df_imovel_clean.drop(columns=['match_encontrado'], inplace=True)
    
    return df_imovel_clean

# Exemplo de uso:
df = df_tombados_filtrado.copy()

# Aplicar a atualização com fuzzy matching
df_imovel_atualizado = atualizar_com_fuzzy_matching(df, df_imovel)

# Mostrar resultados
#print("\nResultado da atualização:")
#print(df_imovel_atualizado)

# Salvar resultado
df_imovel_atualizado.to_csv('imoveis_atualizados_fuzzy.csv', index=False, encoding='utf-8-sig')
```
:::

::: {#9d9cdbbb-eb8c-4de7-ba01-1b024c9c629a .cell .markdown}
### Filtramos o dataframe atualizado
:::

::: {#dcf6f4d3-e2d9-4da7-a8f2-e312e697fd10 .cell .code execution_count="42"}
``` python
df_imovel_nomeado = df_imovel_atualizado[~df_imovel_atualizado['Bem'].str.contains('Imóvel', case=False)]
print(df_imovel_nomeado)
```

::: {.output .stream .stdout}
                                                       Bem   Latitude  Longitude
    60                              Escola Duque de Caxias -21.768333 -43.347222
    61                                     Círculo Militar -21.769722 -43.347778
    62                                  Residência Colucci -21.770556 -43.346944
    63                                     Grupos Centrais -21.771944 -43.347222
    64                  Anexo do Núcleo Histórico da RFFSA -21.758889 -43.343889
    66                   DCE – Antiga Diretoria de Higiene -21.760556 -43.346667
    67                   DCE – Antiga Diretoria de Higiene -21.760556 -43.346667
    68                        Fábrica Bernardo Mascarenhas -21.760556 -43.346667
    69                        Fábrica Bernardo Mascarenhas -21.760000 -43.346389
    70                            Antigo Mercado Municipal -21.754444 -43.352222
    71                                Associação Comercial -21.759722 -43.344167
    72                                Cine Theatro Central -21.761389 -43.348056
    73                                   Fazenda da Tapera -21.738056 -43.360833
    75                              Castelinho dos Bracher -21.764722 -43.344722
    76                              Castelinho dos Bracher -21.764722 -43.344722
    77                              Castelinho dos Bracher -21.764722 -43.345833
    90                                         Cine Palace -21.759167 -43.347222
    91                                         Cine Palace -21.760000 -43.346944
    92                                               CEMIG -21.742222 -43.373889
    93                                Colégio do Carmo GHH -21.756667 -43.354444
    94   Colégio Cristo Redentor, antiga Academia de Co... -21.762500 -43.353056
    95                                     Banco do Brasil -21.760278 -43.346111
    96                               Banco de Crédito Real -21.760556 -43.346667
    97                                         Cine Palace -21.760833 -43.346944
    98                                     Banco do Brasil -21.760556 -43.346667
    99                                     Banco do Brasil -21.761111 -43.348056
    100  Colégio Cristo Redentor, antiga Academia de Co... -21.761389 -43.348056
    101                                    Banco do Brasil -21.761389 -43.348056
    102  Colégio Cristo Redentor, antiga Academia de Co... -21.761389 -43.348056
    103                                    Banco do Brasil -21.761389 -43.348056
    106  Edifício-Sede da Agência da Empresa Brasileira... -21.759444 -43.344444
:::
:::

::: {#4dacb72b-4181-4e89-9c50-58ea2a89ef3a .cell .markdown}
### Unificamos os dataframes
:::

::: {#fdb8c0cd-7f0f-4e17-8474-3630b2b54e09 .cell .code execution_count="43"}
``` python
df_imovel_nomeado = df_imovel_nomeado.iloc[:-1]
df_bens = pd.concat([df_temp, df_imovel_nomeado], axis=0)
print(df_bens.head())
```

::: {.output .stream .stdout}
                                                   Bem   Latitude  Longitude
    1                                 Agência Bradesco -21.761111 -43.348056
    3                                   Alfândega Seca -21.761389 -43.343056
    4                Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
    5  Antiga Estação Ferroviária da Central do Brasil -22.903611 -43.191111
    6      Antiga Estação Ferroviária de Santos Dumont -21.455833 -43.549722
:::
:::

::: {#45405d02-59f5-4619-8f85-782c26791e1b .cell .markdown}
### Convertemos o dataframe final em dicionário
:::

::: {#3411d44a-3b71-4820-88e6-16c4f9c356fe .cell .code execution_count="44"}
``` python
bens = df_bens.to_dict('records')
```
:::

::: {#da692185-c501-4ace-a591-978f495dd7b8 .cell .markdown}
### Visualizamos os dados em um mapa interativo
:::

::: {#8fda445f-d284-4644-ac29-fed881e73973 .cell .code execution_count="45"}
``` python

if not bens:
    print("Nenhum bem foi extraído. Verifique a estrutura da tabela na página.")
else:
    # Criar mapa base centrado em Juiz de Fora
    museu_mapa = folium.Map(location=[-21.7625, -43.35], zoom_start=13)

    # Estrutura GeoJSON para armazenar os bens (apenas para busca)
    obras_geojson = {
        "type": "FeatureCollection",
        "features": []
    }

    # Criar FeatureGroup para os marcadores visíveis
    bens_group = folium.FeatureGroup(name="Bens Tombados", show=True)

    # Preencher o GeoJSON e adicionar marcadores visíveis
    for item in bens:
        nome = item['Bem']
        lat = item['Latitude']
        lon = item['Longitude']

        if lat is not None and lon is not None:
            # Adicionar ao GeoJSON (para busca)
            obras_geojson["features"].append({
                "type": "Feature",
                "properties": {"nome": nome},
                "geometry": {
                    "type": "Point",
                    "coordinates": [lon, lat]
                }
            })

            # Adicionar marcador visível ao FeatureGroup
            folium.Marker(
                location=[lat, lon],
                popup=nome,
                icon=folium.Icon(color="orange", icon="monument", prefix="fa")
            ).add_to(bens_group)

    # Adicionar o FeatureGroup ao mapa
    bens_group.add_to(museu_mapa)

    # Criar camada GeoJSON oculta apenas para busca
    geojson_layer = folium.GeoJson(
        obras_geojson,
        name="GeoJSON Busca",
        style_function=lambda x: {
            'fillOpacity': 0,  # Totalmente transparente
            'opacity': 0,      # Totalmente transparente
            'radius': 0        # Tamanho zero
        },
        marker=folium.Circle(radius=0),  # Marcador invisível
        control=False  # Não aparece no controle de camadas
    ).add_to(museu_mapa)

    # Configurar o plugin de busca na camada GeoJSON oculta
    search_plugin = Search(
        layer=geojson_layer,
        search_label="nome",
        placeholder="Buscar obra...",
        collapsed=False,
        position='topleft'
    ).add_to(museu_mapa)

    # Adicionar controle de camadas
    folium.LayerControl().add_to(museu_mapa)

    # Salvar mapa
    museu_mapa.save("mapa_bens_tombados_jf.html")
    print("Mapa salvo como 'mapa_bens_tombados_jf.html'")
```

::: {.output .stream .stdout}
    Mapa salvo como 'mapa_bens_tombados_jf.html'
:::
:::

::: {#80f873ad-3311-4481-aa16-11061a39ddc3 .cell .markdown}
### Agrupamos os dados em um mapa interativo
:::

::: {#6e7da073-d164-4a93-99e8-0c0cb3031db0 .cell .code execution_count="48"}
``` python
if not bens:
    print("Nenhum bem foi extraído. Verifique a estrutura da tabela na página.")
else:
    # Criar mapa base centrado em Juiz de Fora
    museu_mapa = folium.Map(location=[-21.7625, -43.35], zoom_start=13)

    # Criar um MarkerCluster para agrupar os marcadores
    marker_cluster = MarkerCluster(
        name="Bens Tombados",
        overlay=True,
        control=True,
        options={'maxClusterRadius': 40}
    ).add_to(museu_mapa)

    # Estrutura GeoJSON para armazenar os bens (apenas para busca)
    obras_geojson = {
        "type": "FeatureCollection",
        "features": []
    }

    # Preencher o GeoJSON e adicionar marcadores ao cluster
    for item in bens:
        nome = item['Bem']
        lat = item['Latitude']
        lon = item['Longitude']

        if lat is not None and lon is not None:
            # Adicionar ao GeoJSON (para busca)
            obras_geojson["features"].append({
                "type": "Feature",
                "properties": {"nome": nome},
                "geometry": {
                    "type": "Point",
                    "coordinates": [lon, lat]
                }
            })

            # Adicionar marcador ao cluster
            folium.Marker(
                location=[lat, lon],
                popup=nome,
                icon=folium.Icon(color="orange", icon="monument", prefix="fa")
            ).add_to(marker_cluster)

    # Criar camada GeoJSON oculta apenas para busca
    geojson_layer = folium.GeoJson(
        obras_geojson,
        name="GeoJSON Busca",
        style_function=lambda x: {
            'fillOpacity': 0,  # Totalmente transparente
            'opacity': 0,      # Totalmente transparente
            'radius': 0        # Tamanho zero
        },
        marker=folium.Circle(radius=0),  # Marcador invisível
        control=False  # Não aparece no controle de camadas
    ).add_to(museu_mapa)

    # Configurar o plugin de busca na camada GeoJSON oculta
    search_plugin = Search(
        layer=geojson_layer,
        search_label="nome",
        placeholder="Buscar obra...",
        collapsed=False,
        position='topleft'
    ).add_to(museu_mapa)

    # Adicionar controle de camadas
    folium.LayerControl().add_to(museu_mapa)

    # Salvar mapa
    museu_mapa.save("cluster_bens_tombados_jf.html")
    print("Mapa salvo como 'cluster_bens_tombados_jf.html'")
```

::: {.output .stream .stdout}
    Mapa salvo como 'cluster_bens_tombados_jf.html'
:::
:::

::: {#18a9ff10-4124-4159-9a55-4a0764e5bba3 .cell .markdown}
### Filtramos os pontos dentro do polígono de Juiz de Fora
:::

::: {#ca5ab45b-ca91-4373-a4aa-f3af2d1ec6db .cell .code execution_count="56"}
``` python
# Obter os limites de Juiz de Fora
place = 'Juiz de Fora, MG, Brasil'
gdf = ox.geocoder.geocode_to_gdf(place)

# Extrair as coordenadas da bounding box (bbox)
bbox = gdf.iloc[0][['bbox_west', 'bbox_south', 'bbox_east', 'bbox_north']].values

# Criar o dicionário no formato desejado
JF_BOUNDS = {
    'min_lat': float(bbox[1]),  # bbox_south
    'max_lat': float(bbox[3]),  # bbox_north
    'min_lon': float(bbox[0]),  # bbox_west
    'max_lon': float(bbox[2])   # bbox_east
}

def dentro_dos_limites(lat, lon):
    return (JF_BOUNDS['min_lat'] <= lat <= JF_BOUNDS['max_lat'] and
            JF_BOUNDS['min_lon'] <= lon <= JF_BOUNDS['max_lon'])


if not bens:
    print("Nenhum bem foi extraído. Verifique a estrutura da tabela na página.")
else:
    # Filtrar bens que estão dentro dos limites de JF
    bens_filtrados = [item for item in bens 
                     if item['Latitude'] is not None 
                     and item['Longitude'] is not None
                     and dentro_dos_limites(item['Latitude'], item['Longitude'])]
    
    # Criar mapa base centrado em Juiz de Fora
    museu_mapa = folium.Map(location=[-21.7625, -43.35], zoom_start=13)

    # Criar um MarkerCluster para agrupar os marcadores
    marker_cluster = MarkerCluster(
        name="Bens Tombados",
        overlay=True,
        control=True,
        options={'maxClusterRadius': 40}
    ).add_to(museu_mapa)

    # Estrutura GeoJSON para armazenar os bens (apenas para busca)
    obras_geojson = {
        "type": "FeatureCollection",
        "features": []
    }

    # Preencher o GeoJSON e adicionar marcadores ao cluster
    for item in bens_filtrados:
        nome = item['Bem']
        lat = item['Latitude']
        lon = item['Longitude']

        # Adicionar ao GeoJSON (para busca)
        obras_geojson["features"].append({
            "type": "Feature",
            "properties": {"nome": nome},
            "geometry": {
                "type": "Point",
                "coordinates": [lon, lat]
            }
        })

        # Adicionar marcador ao cluster
        folium.Marker(
            location=[lat, lon],
            popup=nome,
            icon=folium.Icon(color="orange", icon="monument", prefix="fa")
        ).add_to(marker_cluster)

    # Criar camada GeoJSON oculta apenas para busca
    geojson_layer = folium.GeoJson(
        obras_geojson,
        name="GeoJSON Busca",
        style_function=lambda x: {
            'fillOpacity': 0,
            'opacity': 0,
            'radius': 0
        },
        marker=folium.Circle(radius=0),
        control=False
    ).add_to(museu_mapa)

    # Configurar o plugin de busca
    search_plugin = Search(
        layer=geojson_layer,
        search_label="nome",
        placeholder="Buscar obra...",
        collapsed=False,
        position='topleft'
    ).add_to(museu_mapa)

    # Adicionar controle de camadas
    folium.LayerControl().add_to(museu_mapa)

    # Salvar mapa
    museu_mapa.save("cluster_bens_tombados_jf_1.html")
    print(f"Mapa salvo com {len(bens_filtrados)} bens tombados dentro dos limites de JF")
```

::: {.output .stream .stdout}
    Mapa salvo com 117 bens tombados dentro dos limites de JF
:::
:::

::: {#622c7d41-b073-4ed8-be55-bb29793e04e2 .cell .code}
``` python
```
:::
