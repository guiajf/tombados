---
jupyter:
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
  language_info:
    codemirror_mode:
      name: ipython
      version: 3
    file_extension: .py
    mimetype: text/x-python
    name: python
    nbconvert_exporter: python
    pygments_lexer: ipython3
    version: 3.12.3
  nbformat: 4
  nbformat_minor: 5
---

::: {#76e3e982-6efe-4caa-ae28-a6da5db08e71 .cell .markdown}
# Patrimônio histórico de Juiz de Fora
:::

::: {#0d885cac-0f98-4129-82ee-e5834d3507cb .cell .markdown}
Para mapeamento do patrimônio histórico de Juiz de Fora, considerando os
bens tombados, realizamos o download de tabela existente na Wikipedia,
com o auxílio de bibliotecas do Python.
:::

::: {#ad039b0a-d077-49e8-88eb-5031dc14f07d .cell .markdown}
### Importamos as bibliotecas
:::

::: {#bea97f41-f558-4bdc-839a-c501f9a92b76 .cell .code execution_count="26"}
``` python
import requests
import numpy as np
import osmnx as ox
from bs4 import BeautifulSoup
import folium
from folium.plugins import Search, MarkerCluster
import json
```
:::

::: {#df286872-1c13-4d0e-97a6-3f502feae2c6 .cell .markdown}
### Baixamos a lista de bens tombados
:::

::: {#ca14fc82-ad4a-44b0-b9dd-abef445ef73a .cell .code execution_count="2"}
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
:::

::: {#3d16fe6c-8daf-4e78-829d-9b0d98e068c0 .cell .markdown}
### Convertemos a lista em data frame
:::

::: {#152c483d-dd9b-4b26-83ed-86c271674946 .cell .code execution_count="3"}
``` python
import pandas as pd

df = pd.DataFrame(bens)
df.to_csv("bens_tombados_jf.csv", index=False, sep=";", encoding="utf-8")
```
:::

::: {#5b372fe9-4334-4b3a-843e-8d1533d9a790 .cell .code execution_count="4"}
``` python
print(df)
```

::: {.output .stream .stdout}
                                                       Bem   Latitude  Longitude
    0    Acervo documental do "Fundo Câmara Municipal d... -21.755278 -43.344167
    1                                     Agência Bradesco -21.761111 -43.348056
    2                                     Agência Bradesco -21.761111 -43.348056
    3                                       Alfândega Seca -21.761389 -43.343056
    4                    Antiga Diretoria de Higiene – DCE -21.758611 -43.348889
    ..                                                 ...        ...        ...
    135          Prédio da Estação RFFSA e passarela anexa -21.759722 -43.343333
    136  Remanescentes das antigas instalações da Cia M... -21.763056 -43.343056
    137  Remanescentes das antigas instalações da Cia M... -21.763056 -43.343056
    138                                  Tiro de Guerra 17 -21.754444 -43.353056
    139                                      Villa Iracema -21.763333 -43.344444

    [140 rows x 3 columns]
:::
:::

::: {#25ccb245-9c58-4540-91c7-d77a8c884f6e .cell .markdown}
### Inspecionamos o data frame
:::

::: {#ce2cb2a5-3c04-4e85-bd31-d1020b23d482 .cell .code execution_count="5"}
``` python
df.shape
```

::: {.output .execute_result execution_count="5"}
    (140, 3)
:::
:::

::: {#8c85a2b5-3449-4aba-8b7a-f382e4b6c983 .cell .code execution_count="6"}
``` python
print(df.isnull().sum())
```

::: {.output .stream .stdout}
    Bem          0
    Latitude     0
    Longitude    0
    dtype: int64
:::
:::

::: {#e2324b12-a172-4ac0-813c-ba121554ba6b .cell .code execution_count="7"}
``` python
df.isna().sum()
```

::: {.output .execute_result execution_count="7"}
    Bem          0
    Latitude     0
    Longitude    0
    dtype: int64
:::
:::

::: {#2c3346ff-c899-4ddf-8019-f134ffb38b3f .cell .markdown}
### Limpeza de dados
:::

::: {#b3a71e7b-4a4f-4e5c-9312-9792cd793b89 .cell .markdown}
**Removemos os registros com dados em branco**
:::

::: {#93e5a5fa-2fd2-4c0d-9b56-a5882e127025 .cell .code execution_count="8"}
``` python
df = df.replace(['', np.nan], np.nan).dropna()
df
```

::: {.output .execute_result execution_count="8"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Acervo documental do "Fundo Câmara Municipal d...</td>
      <td>-21.755278</td>
      <td>-43.344167</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Agência Bradesco</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Agência Bradesco</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Alfândega Seca</td>
      <td>-21.761389</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Antiga Diretoria de Higiene – DCE</td>
      <td>-21.758611</td>
      <td>-43.348889</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>135</th>
      <td>Prédio da Estação RFFSA e passarela anexa</td>
      <td>-21.759722</td>
      <td>-43.343333</td>
    </tr>
    <tr>
      <th>136</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>137</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>138</th>
      <td>Tiro de Guerra 17</td>
      <td>-21.754444</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>139</th>
      <td>Villa Iracema</td>
      <td>-21.763333</td>
      <td>-43.344444</td>
    </tr>
  </tbody>
</table>
<p>140 rows × 3 columns</p>
</div>
```
:::
:::

::: {#22a12a8b-207d-4f2b-b1a1-d43c6df45f39 .cell .markdown}
**Removemos os dados duplicados**
:::

::: {#eb75b70b-95b4-4741-a652-8eba7b909bdc .cell .code execution_count="9"}
``` python
df = df.drop_duplicates()
df
```

::: {.output .execute_result execution_count="9"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Acervo documental do "Fundo Câmara Municipal d...</td>
      <td>-21.755278</td>
      <td>-43.344167</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Agência Bradesco</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Alfândega Seca</td>
      <td>-21.761389</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Antiga Diretoria de Higiene – DCE</td>
      <td>-21.758611</td>
      <td>-43.348889</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Antiga Estação Ferroviária da Central do Brasil</td>
      <td>-22.903611</td>
      <td>-43.191111</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>135</th>
      <td>Prédio da Estação RFFSA e passarela anexa</td>
      <td>-21.759722</td>
      <td>-43.343333</td>
    </tr>
    <tr>
      <th>136</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>137</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>138</th>
      <td>Tiro de Guerra 17</td>
      <td>-21.754444</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>139</th>
      <td>Villa Iracema</td>
      <td>-21.763333</td>
      <td>-43.344444</td>
    </tr>
  </tbody>
</table>
<p>135 rows × 3 columns</p>
</div>
```
:::
:::

::: {#43b9f734-0bf8-4649-af0b-ce6a5b101a54 .cell .markdown}
**Removemos o primeiro item do dataframe**
:::

::: {#8d47e858-51b3-40a1-8911-c7c8e7172a9a .cell .code execution_count="10"}
``` python
df = df.iloc[1:]
df
```

::: {.output .execute_result execution_count="10"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>Agência Bradesco</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Alfândega Seca</td>
      <td>-21.761389</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Antiga Diretoria de Higiene – DCE</td>
      <td>-21.758611</td>
      <td>-43.348889</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Antiga Estação Ferroviária da Central do Brasil</td>
      <td>-22.903611</td>
      <td>-43.191111</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Antiga Estação Ferroviária de Santos Dumont</td>
      <td>-21.455833</td>
      <td>-43.549722</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>135</th>
      <td>Prédio da Estação RFFSA e passarela anexa</td>
      <td>-21.759722</td>
      <td>-43.343333</td>
    </tr>
    <tr>
      <th>136</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>137</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>138</th>
      <td>Tiro de Guerra 17</td>
      <td>-21.754444</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>139</th>
      <td>Villa Iracema</td>
      <td>-21.763333</td>
      <td>-43.344444</td>
    </tr>
  </tbody>
</table>
<p>134 rows × 3 columns</p>
</div>
```
:::
:::

::: {#230457cb-7b09-4090-a020-0531999d0fef .cell .markdown}
**Criamos o dataframe contendo os bens rotulados**
:::

::: {#6a20328b-5072-4fda-b3ec-b9164d4238a4 .cell .code execution_count="11"}
``` python
df_temp = df[~df['Bem'].str.contains('Imóvel', case=False)]
df_temp
```

::: {.output .execute_result execution_count="11"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>Agência Bradesco</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Alfândega Seca</td>
      <td>-21.761389</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Antiga Diretoria de Higiene – DCE</td>
      <td>-21.758611</td>
      <td>-43.348889</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Antiga Estação Ferroviária da Central do Brasil</td>
      <td>-22.903611</td>
      <td>-43.191111</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Antiga Estação Ferroviária de Santos Dumont</td>
      <td>-21.455833</td>
      <td>-43.549722</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>135</th>
      <td>Prédio da Estação RFFSA e passarela anexa</td>
      <td>-21.759722</td>
      <td>-43.343333</td>
    </tr>
    <tr>
      <th>136</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>137</th>
      <td>Remanescentes das antigas instalações da Cia M...</td>
      <td>-21.763056</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>138</th>
      <td>Tiro de Guerra 17</td>
      <td>-21.754444</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>139</th>
      <td>Villa Iracema</td>
      <td>-21.763333</td>
      <td>-43.344444</td>
    </tr>
  </tbody>
</table>
<p>87 rows × 3 columns</p>
</div>
```
:::
:::

::: {#1f88dbcf-9128-4b84-8914-5b2e3dee3665 .cell .markdown}
### Filtramos os bens sem denominação
:::

::: {#926b61c4-d0a6-458b-86a5-4e5bbb172804 .cell .code execution_count="12"}
``` python
df_imovel = df[df['Bem'].str.contains('Imóvel', case=False)]
df_imovel
```

::: {.output .execute_result execution_count="12"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>60</th>
      <td>Imóvel à Avenida Barão do Rio Branco, nº 3029</td>
      <td>-21.768333</td>
      <td>-43.347222</td>
    </tr>
    <tr>
      <th>61</th>
      <td>Imóvel à Avenida Barão do Rio Branco, nº 3146</td>
      <td>-21.769722</td>
      <td>-43.347778</td>
    </tr>
    <tr>
      <th>62</th>
      <td>Imóvel à Avenida Barão do Rio Branco, nº 3263</td>
      <td>-21.770556</td>
      <td>-43.346944</td>
    </tr>
    <tr>
      <th>63</th>
      <td>Imóvel à Avenida Barão do Rio Branco, nº 3408</td>
      <td>-21.771944</td>
      <td>-43.347222</td>
    </tr>
    <tr>
      <th>64</th>
      <td>Imóvel à Avenida Brasil, nº 2001</td>
      <td>-21.758889</td>
      <td>-43.343889</td>
    </tr>
    <tr>
      <th>65</th>
      <td>Imóvel à Avenida Garibald Campinhos, nº 170</td>
      <td>-21.753333</td>
      <td>-43.344444</td>
    </tr>
    <tr>
      <th>66</th>
      <td>Imóvel à Avenida Getúlio Vargas, nº 434</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>67</th>
      <td>Imóvel à Avenida Getúlio Vargas, nº 438</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>68</th>
      <td>Imóvel à Avenida Getúlio Vargas, nº 438A</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>69</th>
      <td>Imóvel à Avenida Getúlio Vargas, nº 444</td>
      <td>-21.760000</td>
      <td>-43.346389</td>
    </tr>
    <tr>
      <th>70</th>
      <td>Imóvel à Avenida dos Andradas, nº 197</td>
      <td>-21.754444</td>
      <td>-43.352222</td>
    </tr>
    <tr>
      <th>71</th>
      <td>Imóvel à Praça Dr. João Penido, nº 44</td>
      <td>-21.759722</td>
      <td>-43.344167</td>
    </tr>
    <tr>
      <th>72</th>
      <td>Imóvel à Praça João Pessoa s/n</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>73</th>
      <td>Imóvel à Rua Alencar Tristão, nº 236</td>
      <td>-21.738056</td>
      <td>-43.360833</td>
    </tr>
    <tr>
      <th>74</th>
      <td>Imóvel à Rua Antonio Dias, nº 415</td>
      <td>-21.764167</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>75</th>
      <td>Imóvel à Rua Antônio Dias, nº 593</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>76</th>
      <td>Imóvel à Rua Antônio Dias, nº 617</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>77</th>
      <td>Imóvel à Rua Antônio Dias, nº 741</td>
      <td>-21.764722</td>
      <td>-43.345833</td>
    </tr>
    <tr>
      <th>78</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 165</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>79</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 181</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>80</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 195</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>81</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 201</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>82</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 211</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>83</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 217</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>84</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 229</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>85</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 245</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>86</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 249</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>87</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 257</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>88</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 265</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>89</th>
      <td>Imóvel à Rua Barão de Santa Helena, nº 269</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>90</th>
      <td>Imóvel à Rua Batista de Oliveira, nº 377</td>
      <td>-21.759167</td>
      <td>-43.347222</td>
    </tr>
    <tr>
      <th>91</th>
      <td>Imóvel à Rua Batista de Oliveira, nº 483</td>
      <td>-21.760000</td>
      <td>-43.346944</td>
    </tr>
    <tr>
      <th>92</th>
      <td>Imóvel à Rua Bernardo Mascarenhas, nº 1215</td>
      <td>-21.742222</td>
      <td>-43.373889</td>
    </tr>
    <tr>
      <th>93</th>
      <td>Imóvel à Rua Dra. Maria Helena, nº 112</td>
      <td>-21.756667</td>
      <td>-43.354444</td>
    </tr>
    <tr>
      <th>94</th>
      <td>Imóvel à Rua Halfeld, nº 1179</td>
      <td>-21.762500</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>95</th>
      <td>Imóvel à Rua Halfeld, nº 450</td>
      <td>-21.760278</td>
      <td>-43.346111</td>
    </tr>
    <tr>
      <th>96</th>
      <td>Imóvel à Rua Halfeld, nº 504</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>97</th>
      <td>Imóvel à Rua Halfeld, nº 581</td>
      <td>-21.760833</td>
      <td>-43.346944</td>
    </tr>
    <tr>
      <th>98</th>
      <td>Imóvel à Rua Halfeld, nº 599</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>99</th>
      <td>Imóvel à Rua Halfeld, nº 675</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>100</th>
      <td>Imóvel à Rua Halfeld, nº 679</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>101</th>
      <td>Imóvel à Rua Halfeld, nº 703</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>102</th>
      <td>Imóvel à Rua Halfeld, nº 711</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>103</th>
      <td>Imóvel à Rua Halfeld, nº 715</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>104</th>
      <td>Imóvel à Rua Silva Jardim, nº 296</td>
      <td>-21.755278</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>105</th>
      <td>Imóvel à Rua Silva Jardim, nº 306</td>
      <td>-21.755278</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>106</th>
      <td>Imóvel à esquina da Rua Marechal Deodoro com a...</td>
      <td>-21.759444</td>
      <td>-43.344444</td>
    </tr>
  </tbody>
</table>
</div>
```
:::
:::

::: {#5fc55463-7902-4996-b91e-b17272d92c5f .cell .code execution_count="13"}
``` python
df_imovel.shape
```

::: {.output .execute_result execution_count="13"}
    (47, 3)
:::
:::

::: {#addd647c-8ca2-44a9-9a61-30fa65426a07 .cell .markdown}
### Manipulamos arquivo PDF com llm
:::

::: {#426a58ba-22b0-4c25-937a-dac6ab6207ec .cell .code execution_count="14"}
``` python
import os
import pandas as pd
from groq import Groq
from PyPDF2 import PdfReader
import tempfile
import base64

from dotenv import load_dotenv

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

::: {#3b9f6323-5ae1-4a35-b3c0-8d8583eb5092 .cell .code execution_count="15"}
``` python
df_tombados = pd.read_csv("bens_tombados.csv")
df_tombados
```

::: {.output .execute_result execution_count="15"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>endereco</th>
      <th>nome_edificio</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>---</td>
      <td>----------</td>
      <td>-------------</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>Rua Halfeld, s/n</td>
      <td>Edifício do antigo Fórum, atual Câmara Municipal</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>Praça Dr. João Pessoa, s/n</td>
      <td>Cine Theatro Central</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>Parque e Acervo do</td>
      <td>Museu Mariano Procópio</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>Rua Espírito Santo, 467 e Usina de Marmelos</td>
      <td>Remanescentes das antigas instalações da Cia. ...</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>182</th>
      <td>182</td>
      <td>Avenida Getúlio Vargas, 487, 491, 495 e 499</td>
      <td>Hilton Hotel</td>
    </tr>
    <tr>
      <th>183</th>
      <td>183</td>
      <td>NaN</td>
      <td>Morro do Imperador (TV Industrial)</td>
    </tr>
    <tr>
      <th>184</th>
      <td>184</td>
      <td>Rua Marechal Deodoro, 109/111</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>185</th>
      <td>185</td>
      <td>Av. Francisco Valadares, 2745</td>
      <td>Abrigo Santa Helena</td>
    </tr>
    <tr>
      <th>186</th>
      <td>186</td>
      <td>Rua Halfeld, 744</td>
      <td>Edifício Cathoud</td>
    </tr>
  </tbody>
</table>
<p>187 rows × 3 columns</p>
</div>
```
:::
:::

::: {#fdcad449-c2c4-487d-bdba-70bc7e8c88e1 .cell .markdown}
### Removemos os registros em branco
:::

::: {#7ef86d1b-56fd-4a37-b41d-112022d57115 .cell .code execution_count="16"}
``` python
df_tombados_filtrado = df_tombados.dropna()
df_tombados_filtrado
```

::: {.output .execute_result execution_count="16"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>endereco</th>
      <th>nome_edificio</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>---</td>
      <td>----------</td>
      <td>-------------</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>Rua Halfeld, s/n</td>
      <td>Edifício do antigo Fórum, atual Câmara Municipal</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>Praça Dr. João Pessoa, s/n</td>
      <td>Cine Theatro Central</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>Parque e Acervo do</td>
      <td>Museu Mariano Procópio</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>Rua Espírito Santo, 467 e Usina de Marmelos</td>
      <td>Remanescentes das antigas instalações da Cia. ...</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>178</th>
      <td>178</td>
      <td>Rua Osório de Almeida, 642</td>
      <td>Neocolonial verde</td>
    </tr>
    <tr>
      <th>179</th>
      <td>179</td>
      <td>Avenida Rio Branco, 3.103</td>
      <td>Residência Neocolonial</td>
    </tr>
    <tr>
      <th>182</th>
      <td>182</td>
      <td>Avenida Getúlio Vargas, 487, 491, 495 e 499</td>
      <td>Hilton Hotel</td>
    </tr>
    <tr>
      <th>185</th>
      <td>185</td>
      <td>Av. Francisco Valadares, 2745</td>
      <td>Abrigo Santa Helena</td>
    </tr>
    <tr>
      <th>186</th>
      <td>186</td>
      <td>Rua Halfeld, 744</td>
      <td>Edifício Cathoud</td>
    </tr>
  </tbody>
</table>
<p>95 rows × 3 columns</p>
</div>
```
:::
:::

::: {#7cecc0f8-8223-424c-86ac-de6fe9e5f656 .cell .markdown}
### Atribuimos nomes aos registros inomimados
:::

::: {#eb581d87-cbb5-4022-a0d9-e2c5833bacd8 .cell .code execution_count="18"}
``` python
import pandas as pd
import re
from thefuzz import fuzz, process

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
df_funalfa = df_tombados_filtrado.copy()


# Aplicar a atualização com fuzzy matching
df_imovel_atualizado = atualizar_com_fuzzy_matching(df_funalfa, df_imovel)

# Mostrar resultados
print("\nResultado da atualização:")
print(df_imovel_atualizado)

# Salvar resultado
df_imovel_atualizado.to_csv('imoveis_atualizados_fuzzy.csv', index=False, encoding='utf-8-sig')
```

::: {.output .stream .stdout}

    Resultado da atualização:
                                                       Bem   Latitude  Longitude
    60                              Escola Duque de Caxias -21.768333 -43.347222
    61                                     Círculo Militar -21.769722 -43.347778
    62                                  Residência Colucci -21.770556 -43.346944
    63                                     Grupos Centrais -21.771944 -43.347222
    64                  Anexo do Núcleo Histórico da RFFSA -21.758889 -43.343889
    65         Imóvel à Avenida Garibald Campinhos, nº 170 -21.753333 -43.344444
    66                   DCE – Antiga Diretoria de Higiene -21.760556 -43.346667
    67                   DCE – Antiga Diretoria de Higiene -21.760556 -43.346667
    68                        Fábrica Bernardo Mascarenhas -21.760556 -43.346667
    69                        Fábrica Bernardo Mascarenhas -21.760000 -43.346389
    70                            Antigo Mercado Municipal -21.754444 -43.352222
    71                                Associação Comercial -21.759722 -43.344167
    72                                Cine Theatro Central -21.761389 -43.348056
    73                                   Fazenda da Tapera -21.738056 -43.360833
    74                   Imóvel à Rua Antonio Dias, nº 415 -21.764167 -43.343056
    75                              Castelinho dos Bracher -21.764722 -43.344722
    76                              Castelinho dos Bracher -21.764722 -43.344722
    77                              Castelinho dos Bracher -21.764722 -43.345833
    78          Imóvel à Rua Barão de Santa Helena, nº 165 -21.764722 -43.344722
    79          Imóvel à Rua Barão de Santa Helena, nº 181 -21.764722 -43.344722
    80          Imóvel à Rua Barão de Santa Helena, nº 195 -21.764722 -43.344722
    81          Imóvel à Rua Barão de Santa Helena, nº 201 -21.764722 -43.344722
    82          Imóvel à Rua Barão de Santa Helena, nº 211 -21.764722 -43.344722
    83          Imóvel à Rua Barão de Santa Helena, nº 217 -21.764722 -43.344722
    84          Imóvel à Rua Barão de Santa Helena, nº 229 -21.764722 -43.344722
    85          Imóvel à Rua Barão de Santa Helena, nº 245 -21.764722 -43.344722
    86          Imóvel à Rua Barão de Santa Helena, nº 249 -21.764722 -43.344722
    87          Imóvel à Rua Barão de Santa Helena, nº 257 -21.764722 -43.344722
    88          Imóvel à Rua Barão de Santa Helena, nº 265 -21.764722 -43.344722
    89          Imóvel à Rua Barão de Santa Helena, nº 269 -21.764722 -43.344722
    90                                         Cine Palace -21.759167 -43.347222
    91                                         Cine Palace -21.760000 -43.346944
    92                                               CEMIG -21.742222 -43.373889
    93                                Colégio do Carmo GHH -21.756667 -43.354444
    94   Colégio Cristo Redentor, antiga Academia de Co... -21.762500 -43.353056
    95                                     Banco do Brasil -21.760278 -43.346111
    96   Banco de Crédito Real (inclusive Museu e arqui... -21.760556 -43.346667
    97                                         Cine Palace -21.760833 -43.346944
    98                                     Banco do Brasil -21.760556 -43.346667
    99                                     Banco do Brasil -21.761111 -43.348056
    100  Colégio Cristo Redentor, antiga Academia de Co... -21.761389 -43.348056
    101                                    Banco do Brasil -21.761389 -43.348056
    102  Colégio Cristo Redentor, antiga Academia de Co... -21.761389 -43.348056
    103                                    Banco do Brasil -21.761389 -43.348056
    104                  Imóvel à Rua Silva Jardim, nº 296 -21.755278 -43.353056
    105                  Imóvel à Rua Silva Jardim, nº 306 -21.755278 -43.353056
    106  Edifício- Sede da Agência da Empresa Brasileir... -21.759444 -43.344444
:::
:::

::: {#a6c380eb-b0d7-421d-ab6f-1f969ee1b9ae .cell .markdown}
### Removemos os bens que não foram atualizados
:::

::: {#dcf6f4d3-e2d9-4da7-a8f2-e312e697fd10 .cell .code execution_count="19"}
``` python
df_imovel_nomeado = df_imovel_atualizado[~df_imovel_atualizado['Bem'].str.contains('Imóvel', case=False)]
df_imovel_nomeado
```

::: {.output .execute_result execution_count="19"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>60</th>
      <td>Escola Duque de Caxias</td>
      <td>-21.768333</td>
      <td>-43.347222</td>
    </tr>
    <tr>
      <th>61</th>
      <td>Círculo Militar</td>
      <td>-21.769722</td>
      <td>-43.347778</td>
    </tr>
    <tr>
      <th>62</th>
      <td>Residência Colucci</td>
      <td>-21.770556</td>
      <td>-43.346944</td>
    </tr>
    <tr>
      <th>63</th>
      <td>Grupos Centrais</td>
      <td>-21.771944</td>
      <td>-43.347222</td>
    </tr>
    <tr>
      <th>64</th>
      <td>Anexo do Núcleo Histórico da RFFSA</td>
      <td>-21.758889</td>
      <td>-43.343889</td>
    </tr>
    <tr>
      <th>66</th>
      <td>DCE – Antiga Diretoria de Higiene</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>67</th>
      <td>DCE – Antiga Diretoria de Higiene</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>68</th>
      <td>Fábrica Bernardo Mascarenhas</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>69</th>
      <td>Fábrica Bernardo Mascarenhas</td>
      <td>-21.760000</td>
      <td>-43.346389</td>
    </tr>
    <tr>
      <th>70</th>
      <td>Antigo Mercado Municipal</td>
      <td>-21.754444</td>
      <td>-43.352222</td>
    </tr>
    <tr>
      <th>71</th>
      <td>Associação Comercial</td>
      <td>-21.759722</td>
      <td>-43.344167</td>
    </tr>
    <tr>
      <th>72</th>
      <td>Cine Theatro Central</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>73</th>
      <td>Fazenda da Tapera</td>
      <td>-21.738056</td>
      <td>-43.360833</td>
    </tr>
    <tr>
      <th>75</th>
      <td>Castelinho dos Bracher</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>76</th>
      <td>Castelinho dos Bracher</td>
      <td>-21.764722</td>
      <td>-43.344722</td>
    </tr>
    <tr>
      <th>77</th>
      <td>Castelinho dos Bracher</td>
      <td>-21.764722</td>
      <td>-43.345833</td>
    </tr>
    <tr>
      <th>90</th>
      <td>Cine Palace</td>
      <td>-21.759167</td>
      <td>-43.347222</td>
    </tr>
    <tr>
      <th>91</th>
      <td>Cine Palace</td>
      <td>-21.760000</td>
      <td>-43.346944</td>
    </tr>
    <tr>
      <th>92</th>
      <td>CEMIG</td>
      <td>-21.742222</td>
      <td>-43.373889</td>
    </tr>
    <tr>
      <th>93</th>
      <td>Colégio do Carmo GHH</td>
      <td>-21.756667</td>
      <td>-43.354444</td>
    </tr>
    <tr>
      <th>94</th>
      <td>Colégio Cristo Redentor, antiga Academia de Co...</td>
      <td>-21.762500</td>
      <td>-43.353056</td>
    </tr>
    <tr>
      <th>95</th>
      <td>Banco do Brasil</td>
      <td>-21.760278</td>
      <td>-43.346111</td>
    </tr>
    <tr>
      <th>96</th>
      <td>Banco de Crédito Real (inclusive Museu e arqui...</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>97</th>
      <td>Cine Palace</td>
      <td>-21.760833</td>
      <td>-43.346944</td>
    </tr>
    <tr>
      <th>98</th>
      <td>Banco do Brasil</td>
      <td>-21.760556</td>
      <td>-43.346667</td>
    </tr>
    <tr>
      <th>99</th>
      <td>Banco do Brasil</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>100</th>
      <td>Colégio Cristo Redentor, antiga Academia de Co...</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>101</th>
      <td>Banco do Brasil</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>102</th>
      <td>Colégio Cristo Redentor, antiga Academia de Co...</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>103</th>
      <td>Banco do Brasil</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>106</th>
      <td>Edifício- Sede da Agência da Empresa Brasileir...</td>
      <td>-21.759444</td>
      <td>-43.344444</td>
    </tr>
  </tbody>
</table>
</div>
```
:::
:::

::: {#d7c97bfe-ef16-4d35-8c99-6d6a94ade091 .cell .markdown}
### Removemos os bens duplicados
:::

::: {#fdb8c0cd-7f0f-4e17-8474-3630b2b54e09 .cell .code execution_count="20"}
``` python
df_imovel_nomeado = df_imovel_nomeado.iloc[:-1]
df_bens = pd.concat([df_temp, df_imovel_nomeado], axis=0)
df_bens
```

::: {.output .execute_result execution_count="20"}
```{=html}
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bem</th>
      <th>Latitude</th>
      <th>Longitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>Agência Bradesco</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Alfândega Seca</td>
      <td>-21.761389</td>
      <td>-43.343056</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Antiga Diretoria de Higiene – DCE</td>
      <td>-21.758611</td>
      <td>-43.348889</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Antiga Estação Ferroviária da Central do Brasil</td>
      <td>-22.903611</td>
      <td>-43.191111</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Antiga Estação Ferroviária de Santos Dumont</td>
      <td>-21.455833</td>
      <td>-43.549722</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>99</th>
      <td>Banco do Brasil</td>
      <td>-21.761111</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>100</th>
      <td>Colégio Cristo Redentor, antiga Academia de Co...</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>101</th>
      <td>Banco do Brasil</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>102</th>
      <td>Colégio Cristo Redentor, antiga Academia de Co...</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
    <tr>
      <th>103</th>
      <td>Banco do Brasil</td>
      <td>-21.761389</td>
      <td>-43.348056</td>
    </tr>
  </tbody>
</table>
<p>117 rows × 3 columns</p>
</div>
```
:::
:::

::: {#0803f71c-8ca5-4531-a416-5aaa74415e62 .cell .code execution_count="21"}
``` python
df_temp.shape
```

::: {.output .execute_result execution_count="21"}
    (87, 3)
:::
:::

::: {#040cee7f-68c2-4edf-be2e-a12ba043edb0 .cell .code execution_count="22"}
``` python
df_imovel_nomeado.shape
```

::: {.output .execute_result execution_count="22"}
    (30, 3)
:::
:::

::: {#7d2b0f99-e12d-4704-9ce0-ebd75d8c9398 .cell .code execution_count="23"}
``` python
bens = df_bens.to_dict('records')
```
:::

::: {#ca08128a-acba-4a8d-afc1-4d510c5e65b9 .cell .markdown}
### Visualizamos os bens tombados em um mapa interativo
:::

::: {#8fda445f-d284-4644-ac29-fed881e73973 .cell .code execution_count="24"}
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

::: {#1d886fe6-40de-4f2f-9f63-3a80440c71b3 .cell .markdown}
### Agrupamos os bens tombados
:::

::: {#6e7da073-d164-4a93-99e8-0c0cb3031db0 .cell .code execution_count="27"}
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

::: {#ca5ab45b-ca91-4373-a4aa-f3af2d1ec6db .cell .code execution_count="28"}
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
    Mapa salvo com 115 bens tombados dentro dos limites de JF
:::
:::

::: {#622c7d41-b073-4ed8-be55-bb29793e04e2 .cell .code}
``` python
```
:::
