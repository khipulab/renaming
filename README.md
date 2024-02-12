[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.6908292.svg)](https://doi.org/10.5281/zenodo.6908292)


## Supplemental Code for "A New Naming Convention for Andean Khipus"

The code in this repository can be used to reproduce the renaming procedure described in [Brezine, Clindaniel, Ghezzi, Hyland and Medrano (2024)](https://doi.org/10.1017/laq.2023.71) "A New Naming Convention for Andean Khipus."

The code is written in Python 3.9.12 and all of its dependencies can be installed by running the following in the terminal (with the `requirements.txt` file included in this repository):

```bash
pip install -r requirements.txt
```

Once Python is installed, download the current version of the OKR (v1.0.1 -- either via GitHub or Zenodo) and connect to the database via Python's `sqlite3` module.


```python
! git clone https://github.com/khipulab/open-khipu-repository.git -b v1.0.1 --single-branch --quiet
```


```python
import sqlite3
import pandas as pd

conn = sqlite3.connect('open-khipu-repository/khipu.db')

def delete_khipu(khipu_to_delete, con):
    '''
    Description: Deletes khipu corresponding to `khipu_to_delete`
                 from db (erasing all entries associated with it
                 in all relevant tables)

    Input: khipu_to_delete (khipu_id in db; string of 7 integers), 
           SQLite3 connection object
           
    Returns: Nothing (prints khipu that has been deleted)
    '''
    cur = conn.cursor()
    script = f'''
    DELETE FROM knot
    WHERE
        cord_id
        IN
        (SELECT cord_id
        FROM cord
        WHERE khipu_id = {khipu_to_delete}
        );

    DELETE FROM knot_cluster
    WHERE
        cord_id
        IN
        (SELECT cord_id
        FROM cord
        WHERE khipu_id = {khipu_to_delete}
        );
        
    DELETE FROM cord
    WHERE khipu_id = {khipu_to_delete};

    DELETE FROM cord_cluster
    WHERE khipu_id = {khipu_to_delete};

    DELETE FROM ascher_cord_color
    WHERE khipu_id = {khipu_to_delete};

    DELETE FROM khipu_main
    WHERE khipu_id = {khipu_to_delete};

    DELETE FROM primary_cord
    WHERE khipu_id = {khipu_to_delete};
    '''

    cur.executescript(script)
    con.commit()
    
    print(f'khipu_id {khipu_to_delete} Deleted')
```

First, we drop the following khipu records that are duplicates of other khipus in the database, or are blank/incomplete records (none of these received an OKR_NUM).


```python
# Drop khipu records that are not listed in translation table
khipu_ids_to_delete = [1000594, 1000364, 1000484, 1000498]
for khipu_id in khipu_ids_to_delete:
    delete_khipu(khipu_id, conn)
```

    khipu_id 1000594 Deleted
    khipu_id 1000364 Deleted
    khipu_id 1000484 Deleted
    khipu_id 1000498 Deleted
    

Then, we can add the OKR_NUM column to the KHIPU_MAIN table:


```python
add_okr_num_col = 'ALTER TABLE khipu_main ADD COLUMN OKR_NUM LONGTEXT'
cur = conn.cursor()
cur.execute(add_okr_num_col)
conn.commit()
```

...and use the translation table (rewritten as a CSV file in this repository called `renaming.csv`) in the Supplemental Appendix to map an OKR_NUM onto each INVESTIGATOR_NUM:


```python
df = pd.read_csv('renaming.csv')
df.loc[:, 'cur_num'] = df.cur_num.apply(lambda x: [i.strip() for i in x.split(",")])
df_explode = df.explode('cur_num') \
               .reset_index(drop=True)
df_explode.head()
```



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>okr_num</th>
      <th>cur_num</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>KH0001</td>
      <td>LL01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>KH0001</td>
      <td>UR176</td>
    </tr>
    <tr>
      <th>2</th>
      <td>KH0002</td>
      <td>AS001</td>
    </tr>
    <tr>
      <th>3</th>
      <td>KH0003</td>
      <td>AS002</td>
    </tr>
    <tr>
      <th>4</th>
      <td>KH0004</td>
      <td>AS003</td>
    </tr>
  </tbody>
</table>




```python
for okr_num, investigator_num in df_explode.to_records(index=False):
    add_okr_num = f"UPDATE khipu_main SET okr_num='{okr_num}' where investigator_num='{investigator_num}'"
    cur = conn.cursor()
    cur.execute(add_okr_num)
    conn.commit()
```


```python
# check to make sure it worked
pd.read_sql_query('''
                  SELECT okr_num, group_concat(investigator_num,"/")
                  FROM khipu_main
                  GROUP BY okr_num
                  ''', conn).head()
```




<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>OKR_NUM</th>
      <th>group_concat(investigator_num,"/")</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>KH0001</td>
      <td>LL01/UR176</td>
    </tr>
    <tr>
      <th>1</th>
      <td>KH0011</td>
      <td>AS010</td>
    </tr>
    <tr>
      <th>2</th>
      <td>KH0012</td>
      <td>AS011</td>
    </tr>
    <tr>
      <th>3</th>
      <td>KH0013</td>
      <td>AS012</td>
    </tr>
    <tr>
      <th>4</th>
      <td>KH0014</td>
      <td>AS013</td>
    </tr>
  </tbody>
</table>




At this point, we need to drop the remaining duplicate entries of khipus (there are 13 duplicate pairings, as per the Supplemental Appendix) and then add the INVESTIGATOR_NUM values that we drop to the canonical reading of each khipu (e.g. the reading that is most complete/most recent). First, let's gather the KHIPU_ID values for the khipus we want to drop and the ones that will serve as our canonical khipu recordings:


```python
khipu_id_drop = pd.read_sql_query('''
                                    SELECT khipu_id
                                    FROM khipu_main
                                    WHERE investigator_num IN 
                                    ('AS070','AS030','UR044','AS208',
                                    'UR115','UR036','AS056','LL01',
                                    'AS181','AS068','AS047','AS038',
                                    'AS046')
                                    ''',
                                    conn
)

khipu_id_keep = pd.read_sql_query('''
                                    SELECT khipu_id, okr_num
                                    FROM khipu_main
                                    WHERE investigator_num IN 
                                    ('UR035','UR043','UR1031','UR083',
                                    'UR126','UR133','UR163','UR176',
                                    'UR236','UR281','HP035','HP036',
                                    'HP041')
                                    ''',
                                    conn
)
```

Then, we can delete the khipus that are duplicates...


```python
for khipu_id in khipu_id_drop.values.flatten():
    delete_khipu(khipu_id, conn) #13 deleted
```

    khipu_id 1000016 Deleted
    khipu_id 1000095 Deleted
    khipu_id 1000102 Deleted
    khipu_id 1000153 Deleted
    khipu_id 1000175 Deleted
    khipu_id 1000181 Deleted
    khipu_id 1000202 Deleted
    khipu_id 1000237 Deleted
    khipu_id 1000281 Deleted
    khipu_id 1000286 Deleted
    khipu_id 1000334 Deleted
    khipu_id 1000360 Deleted
    khipu_id 1000471 Deleted
    

And update the INVESTIGATOR_NUM entries for the khipus that we are keeping as canonical readings:


```python
df.loc[:, 'agg_inv_num'] = df.cur_num.apply(lambda x: '/'.join(x))
agg_inv_num_df = khipu_id_keep.merge(df, left_on='OKR_NUM', right_on='okr_num')
agg_inv_num_df[['okr_num', 'agg_inv_num']]
```





<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>okr_num</th>
      <th>agg_inv_num</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>KH0033</td>
      <td>AS031/UR1031/UR044</td>
    </tr>
    <tr>
      <th>1</th>
      <td>KH0032</td>
      <td>AS030/UR043</td>
    </tr>
    <tr>
      <th>2</th>
      <td>KH0083</td>
      <td>AS070/UR035</td>
    </tr>
    <tr>
      <th>3</th>
      <td>KH0268</td>
      <td>UR036/UR133</td>
    </tr>
    <tr>
      <th>4</th>
      <td>KH0351</td>
      <td>UR115/UR126</td>
    </tr>
    <tr>
      <th>5</th>
      <td>KH0067</td>
      <td>AS056/UR163</td>
    </tr>
    <tr>
      <th>6</th>
      <td>KH0228</td>
      <td>AS208/UR083</td>
    </tr>
    <tr>
      <th>7</th>
      <td>KH0001</td>
      <td>LL01/UR176</td>
    </tr>
    <tr>
      <th>8</th>
      <td>KH0198</td>
      <td>AS181/UR236</td>
    </tr>
    <tr>
      <th>9</th>
      <td>KH0058</td>
      <td>AS047/HP035</td>
    </tr>
    <tr>
      <th>10</th>
      <td>KH0049</td>
      <td>AS038/HP036</td>
    </tr>
    <tr>
      <th>11</th>
      <td>KH0057</td>
      <td>AS046/HP041</td>
    </tr>
    <tr>
      <th>12</th>
      <td>KH0081</td>
      <td>AS068/UR281</td>
    </tr>
  </tbody>
</table>





```python
agg_inv_array = agg_inv_num_df[['KHIPU_ID', 'agg_inv_num']].to_records(index=False)

for khipu_id, agg_inv_num in agg_inv_array:
    update_inv_num = f"UPDATE khipu_main SET investigator_num='{agg_inv_num}' where khipu_id='{khipu_id}'"
    cur = conn.cursor()
    cur.execute(update_inv_num)
    conn.commit()
```

Confirm that there are 619 unique `okr_num` entries and that the investigator_num updates were performed correctly:


```python
khipu_main = pd.read_sql_query(f'''SELECT *
                                   FROM khipu_main
                                ''', conn)

print("There are {} unique OKR_NUM entries".format(len(khipu_main.OKR_NUM)))

khipu_main.loc[khipu_main.KHIPU_ID.isin(agg_inv_num_df.KHIPU_ID), ['OKR_NUM','INVESTIGATOR_NUM']]
```

    There are 619 unique OKR_NUM entries
    





<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>OKR_NUM</th>
      <th>INVESTIGATOR_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>95</th>
      <td>KH0033</td>
      <td>AS031/UR1031/UR044</td>
    </tr>
    <tr>
      <th>99</th>
      <td>KH0032</td>
      <td>AS030/UR043</td>
    </tr>
    <tr>
      <th>265</th>
      <td>KH0083</td>
      <td>AS070/UR035</td>
    </tr>
    <tr>
      <th>297</th>
      <td>KH0268</td>
      <td>UR036/UR133</td>
    </tr>
    <tr>
      <th>306</th>
      <td>KH0351</td>
      <td>UR115/UR126</td>
    </tr>
    <tr>
      <th>359</th>
      <td>KH0067</td>
      <td>AS056/UR163</td>
    </tr>
    <tr>
      <th>362</th>
      <td>KH0228</td>
      <td>AS208/UR083</td>
    </tr>
    <tr>
      <th>455</th>
      <td>KH0001</td>
      <td>LL01/UR176</td>
    </tr>
    <tr>
      <th>507</th>
      <td>KH0198</td>
      <td>AS181/UR236</td>
    </tr>
    <tr>
      <th>524</th>
      <td>KH0058</td>
      <td>AS047/HP035</td>
    </tr>
    <tr>
      <th>525</th>
      <td>KH0049</td>
      <td>AS038/HP036</td>
    </tr>
    <tr>
      <th>529</th>
      <td>KH0057</td>
      <td>AS046/HP041</td>
    </tr>
    <tr>
      <th>605</th>
      <td>KH0081</td>
      <td>AS068/UR281</td>
    </tr>
  </tbody>
</table>



Finally, there are several updates to provenance and museum numbers that need to be made as corrections to the original readings in the DB.

`provenance` updates (from Supplemental Appendix)


```python
provenance_updates = [
('KH0085', 'Rancho San Juan, Ica Valley'),
('KH0086', 'Rancho San Juan, Ica Valley')
]

for okr_num, new_provenance in provenance_updates:
    update_provenance = f"UPDATE khipu_main SET provenance='{new_provenance}' where okr_num='{okr_num}'"
    cur = conn.cursor()
    cur.execute(update_provenance)
    conn.commit()
```


```python
khipu_main = pd.read_sql_query(f'''SELECT *
                                   FROM khipu_main
                                   WHERE okr_num IN
                                   {tuple(i[0] for i in provenance_updates)}
                                ''', conn)

khipu_main[['OKR_NUM', 'PROVENANCE']]
```





<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>OKR_NUM</th>
      <th>PROVENANCE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>KH0086</td>
      <td>Rancho San Juan, Ica Valley</td>
    </tr>
    <tr>
      <th>1</th>
      <td>KH0085</td>
      <td>Rancho San Juan, Ica Valley</td>
    </tr>
  </tbody>
</table>





`museum_num` updates (from Supplemental Appendix):


```python
museum_num_updates = [
('KH0120', 'VA24370(A)'),
('KH0121', 'VA24370(B)'),
('KH0142', 'VA63042(A)'),
('KH0143', 'VA63042(B)'),
('KH0189', 'VA16145(A)'),
('KH0190', 'VA16145(B)'),
('KH0193', 'VA37859(A)'),
('KH0194', 'VA37859(B)'),
('KH0197', 'VA66832'),
('KH0264', 'TM 4/5446'),
('KH0265', 'TM 4/5446'),
('KH0273', '32.30.30/53(A)'),
('KH0348', '1924.18.0001'),
('KH0349', '1931.37.0001'),
('KH0437', 'VA42597(A)'),
('KH0438', 'VA42597(B)'),
('KH0441', 'VA47114c(A)'),
('KH0442', 'VA47114c(B)'),
('KH0443', 'VA47114c(C)'),
('KH0447', 'VA16141(A)'),
('KH0448', 'VA16141(B)'),
('KH0450', 'VA42508(A)'),
('KH0451', 'VA42508(B)'),
('KH0458', 'VA47114b'),
('KH0463', 'VA44677a(A)'),
('KH0464', 'VA44677a(B)'),
('KH0468', 'VA63038(A)'),
('KH0469', 'VA63038(B)'),
('KH0478', 'VA42607(A)'),
('KH0479', 'VA42607(B)'),
('KH0480', 'VA42607(C)'),
('KH0481', 'VA42607(D)'),
('KH0484', 'VA42578i28'),
('KH0535', 'MSP 1389/RN 43370'),
('KH0558', 'MSP 1422/RN 43403'),
('KH0567', 'MNAAHP 4202'),
('KH0587', 'MNAAHP 30564'),
('KH0588', 'B397/T41299.22'),
('KH0589', 'B376/T41299.23'),
('KH0590', 'B388/T41299.24'),
('KH0591', 'B378/T41299.25'),
('KH0592', 'B377/T41299.26'),
('KH0593', 'B384/T41299.27'),
('KH0594', 'B372/T41299.28'),
('KH0595', 'B367/T41299.29'),
('KH0596', 'B366/T41299.30'),
('KH0597', 'B374/T41299.31'),
('KH0598', 'B375/T41299.32'),
('KH0599', 'B391/T41299.20'),
('KH0600', 'B369/T41299.33.A-B'),
('KH0601', 'B399/T41299.34'),
('KH0602', 'B373/T41299.18'),
('KH0603', 'B383&B383A/T41299.35.A-B'),
('KH0604', 'B395/T41299.36'),
('KH0605', 'B382/T41299.37'),
('KH0606', 'B371/T41299.38'),
('KH0405', '41.0/1550, B/3453A')
]

for okr_num, new_museum_num in museum_num_updates:
    update_museum_num = f"UPDATE khipu_main SET museum_num='{new_museum_num}' where okr_num='{okr_num}'"
    cur = conn.cursor()
    cur.execute(update_museum_num)
    conn.commit()
```


```python
khipu_main = pd.read_sql_query(f'''SELECT *
                                   FROM khipu_main
                                   WHERE okr_num IN
                                   {tuple(i[0] for i in museum_num_updates)}
                                ''', conn)

khipu_main[['OKR_NUM', 'MUSEUM_NUM']]
```





<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>OKR_NUM</th>
      <th>MUSEUM_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>KH0120</td>
      <td>VA24370(A)</td>
    </tr>
    <tr>
      <th>1</th>
      <td>KH0121</td>
      <td>VA24370(B)</td>
    </tr>
    <tr>
      <th>2</th>
      <td>KH0190</td>
      <td>VA16145(B)</td>
    </tr>
    <tr>
      <th>3</th>
      <td>KH0142</td>
      <td>VA63042(A)</td>
    </tr>
    <tr>
      <th>4</th>
      <td>KH0143</td>
      <td>VA63042(B)</td>
    </tr>
    <tr>
      <th>5</th>
      <td>KH0273</td>
      <td>32.30.30/53(A)</td>
    </tr>
    <tr>
      <th>6</th>
      <td>KH0189</td>
      <td>VA16145(A)</td>
    </tr>
    <tr>
      <th>7</th>
      <td>KH0193</td>
      <td>VA37859(A)</td>
    </tr>
    <tr>
      <th>8</th>
      <td>KH0194</td>
      <td>VA37859(B)</td>
    </tr>
    <tr>
      <th>9</th>
      <td>KH0197</td>
      <td>VA66832</td>
    </tr>
    <tr>
      <th>10</th>
      <td>KH0349</td>
      <td>1931.37.0001</td>
    </tr>
    <tr>
      <th>11</th>
      <td>KH0264</td>
      <td>TM 4/5446</td>
    </tr>
    <tr>
      <th>12</th>
      <td>KH0265</td>
      <td>TM 4/5446</td>
    </tr>
    <tr>
      <th>13</th>
      <td>KH0348</td>
      <td>1924.18.0001</td>
    </tr>
    <tr>
      <th>14</th>
      <td>KH0535</td>
      <td>MSP 1389/RN 43370</td>
    </tr>
    <tr>
      <th>15</th>
      <td>KH0558</td>
      <td>MSP 1422/RN 43403</td>
    </tr>
    <tr>
      <th>16</th>
      <td>KH0589</td>
      <td>B376/T41299.23</td>
    </tr>
    <tr>
      <th>17</th>
      <td>KH0591</td>
      <td>B378/T41299.25</td>
    </tr>
    <tr>
      <th>18</th>
      <td>KH0590</td>
      <td>B388/T41299.24</td>
    </tr>
    <tr>
      <th>19</th>
      <td>KH0592</td>
      <td>B377/T41299.26</td>
    </tr>
    <tr>
      <th>20</th>
      <td>KH0593</td>
      <td>B384/T41299.27</td>
    </tr>
    <tr>
      <th>21</th>
      <td>KH0594</td>
      <td>B372/T41299.28</td>
    </tr>
    <tr>
      <th>22</th>
      <td>KH0595</td>
      <td>B367/T41299.29</td>
    </tr>
    <tr>
      <th>23</th>
      <td>KH0596</td>
      <td>B366/T41299.30</td>
    </tr>
    <tr>
      <th>24</th>
      <td>KH0597</td>
      <td>B374/T41299.31</td>
    </tr>
    <tr>
      <th>25</th>
      <td>KH0598</td>
      <td>B375/T41299.32</td>
    </tr>
    <tr>
      <th>26</th>
      <td>KH0599</td>
      <td>B391/T41299.20</td>
    </tr>
    <tr>
      <th>27</th>
      <td>KH0601</td>
      <td>B399/T41299.34</td>
    </tr>
    <tr>
      <th>28</th>
      <td>KH0602</td>
      <td>B373/T41299.18</td>
    </tr>
    <tr>
      <th>29</th>
      <td>KH0603</td>
      <td>B383&amp;B383A/T41299.35.A-B</td>
    </tr>
    <tr>
      <th>30</th>
      <td>KH0604</td>
      <td>B395/T41299.36</td>
    </tr>
    <tr>
      <th>31</th>
      <td>KH0605</td>
      <td>B382/T41299.37</td>
    </tr>
    <tr>
      <th>32</th>
      <td>KH0606</td>
      <td>B371/T41299.38</td>
    </tr>
    <tr>
      <th>33</th>
      <td>KH0405</td>
      <td>41.0/1550, B/3453A</td>
    </tr>
    <tr>
      <th>34</th>
      <td>KH0437</td>
      <td>VA42597(A)</td>
    </tr>
    <tr>
      <th>35</th>
      <td>KH0438</td>
      <td>VA42597(B)</td>
    </tr>
    <tr>
      <th>36</th>
      <td>KH0441</td>
      <td>VA47114c(A)</td>
    </tr>
    <tr>
      <th>37</th>
      <td>KH0442</td>
      <td>VA47114c(B)</td>
    </tr>
    <tr>
      <th>38</th>
      <td>KH0443</td>
      <td>VA47114c(C)</td>
    </tr>
    <tr>
      <th>39</th>
      <td>KH0447</td>
      <td>VA16141(A)</td>
    </tr>
    <tr>
      <th>40</th>
      <td>KH0448</td>
      <td>VA16141(B)</td>
    </tr>
    <tr>
      <th>41</th>
      <td>KH0450</td>
      <td>VA42508(A)</td>
    </tr>
    <tr>
      <th>42</th>
      <td>KH0451</td>
      <td>VA42508(B)</td>
    </tr>
    <tr>
      <th>43</th>
      <td>KH0458</td>
      <td>VA47114b</td>
    </tr>
    <tr>
      <th>44</th>
      <td>KH0463</td>
      <td>VA44677a(A)</td>
    </tr>
    <tr>
      <th>45</th>
      <td>KH0464</td>
      <td>VA44677a(B)</td>
    </tr>
    <tr>
      <th>46</th>
      <td>KH0468</td>
      <td>VA63038(A)</td>
    </tr>
    <tr>
      <th>47</th>
      <td>KH0469</td>
      <td>VA63038(B)</td>
    </tr>
    <tr>
      <th>48</th>
      <td>KH0478</td>
      <td>VA42607(A)</td>
    </tr>
    <tr>
      <th>49</th>
      <td>KH0479</td>
      <td>VA42607(B)</td>
    </tr>
    <tr>
      <th>50</th>
      <td>KH0481</td>
      <td>VA42607(D)</td>
    </tr>
    <tr>
      <th>51</th>
      <td>KH0480</td>
      <td>VA42607(C)</td>
    </tr>
    <tr>
      <th>52</th>
      <td>KH0484</td>
      <td>VA42578i28</td>
    </tr>
    <tr>
      <th>53</th>
      <td>KH0567</td>
      <td>MNAAHP 4202</td>
    </tr>
    <tr>
      <th>54</th>
      <td>KH0587</td>
      <td>MNAAHP 30564</td>
    </tr>
    <tr>
      <th>55</th>
      <td>KH0588</td>
      <td>B397/T41299.22</td>
    </tr>
  </tbody>
</table>



