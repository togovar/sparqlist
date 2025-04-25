# Mouse strain ID to mouse strain attributes

Return attributes of a mouse strain for strain_id.

## Parameters

* `strain_id` mouse strain ID (to name)
  * example: msmv4_sq,jf1v3  or all
* `strain` mouse strain name (to ID)
  * example: MSMv4,129S1/SvImJ or all

## `result`

```javascript
async ({strain_id, strain}) => {

let id2strain = {
  "refGenome": {
    "strain": "C57BL/6J",
    "id": "refGenome",
    "category": "reference",
    "source": ""
  },
  "msmv4_sq": {
    "strain": "MSMv4",
    "id": "msmv4_sq",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00209"
  },
  "jf1v3": {
    "strain": "JF1v3",
    "id": "jf1v3",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00639"
  },
  "kjrv1": {
    "strain": "KJRv1",
    "id": "kjrv1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00655"
  },
  "swnv1": {
    "strain": "SWNv1",
    "id": "swnv1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00654"
  },
  "chdv1": {
    "strain": "CHDv1",
    "id": "chdv1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00738"
  },
  "njlv1": {
    "strain": "NJLv1",
    "id": "njlv1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00207"
  },
  "blg2v1": {
    "strain": "BLG2v1",
    "id": "blg2v1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00653"
  },
  "hmiv1": {
    "strain": "HMIv1",
    "id": "hmiv1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00657"
  },
  "bfmv1": {
    "strain": "BFMv1",
    "id": "bfmv1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00659"
  },
  "pgn2v1": {
    "strain": "PGN2v1",
    "id": "pgn2v1",
    "category": "mogplus21",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00667"
  },
  "MSM": {
    "strain": "MSMagv1.1",
    "id": "MSM",
    "category": "mogplus3",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00209"
  },
  "JF1": {
    "strain": "JF1agv1.1",
    "id": "JF1",
    "category": "mogplus3",
    "source": "https://knowledge.brc.riken.jp/resource/animal/card?brc_no=RBRC00639"
  },
  "129P2_OlaHsd": {
    "strain": "129P2/OlaHsd",
    "id": "129P2_OlaHsd",
    "category": "other",
    "source": ""
  },
  "129S1_SvImJ": {
    "strain": "129S1/SvImJ",
    "id": "129S1_SvImJ",
    "category": "other",
    "source": "https://www.jax.org/strain/002448"
  },
  "129S5SvEvBrd": {
    "strain": "129S5SvEvBrd",
    "id": "129S5SvEvBrd",
    "category": "other",
    "source": ""
  },
  "A_J": {
    "strain": "A/J",
    "id": "A_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000646"
  },
  "AKR_J": {
    "strain": "AKR/J",
    "id": "AKR_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000648"
  },
  "B10.RIII-H2r H2-T18b/(71NS)SnJ": {
    "strain": "B10.RIII-H2r H2-T18b/(71NS)SnJ",
    "id": "B10.RIII",
    "category": "other",
    "source": ""
  },
  "BALB_cByJ": {
    "strain": "BALB/cByJ",
    "id": "BALB_cByJ",
    "category": "other",
    "source": ""
  },
  "BALB_cJ": {
    "strain": "BALB/cJ",
    "id": "BALB_cJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000651"
  },
  "BTBR_T+_Itpr3tf_J": {
    "strain": "BTBR T+Itpr3tf/J",
    "id": "BTBR_T+_Itpr3tf_J",
    "category": "other",
    "source": "https://www.jax.org/strain/002282"
  },
  "BUB_BnJ": {
    "strain": "BUB/BnJ",
    "id": "BUB_BnJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000653"
  },
  "C3H_HeH": {
    "strain": "C3H/HeH",
    "id": "C3H_HeH",
    "category": "othe",
    "source": ""
  },
  "C3H_HeJ": {
    "strain": "C3H/HeJ",
    "id": "C3H_HeJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000659"
  },
  "C57BL_10J": {
    "strain": "C57BL/10J",
    "id": "C57BL_10J",
    "category": "other",
    "source": "https://www.jax.org/strain/000665"
  },
  "C57BL_10SnJ": {
    "strain": "C57BL/10SnJ",
    "id": "C57BL_10SnJ",
    "category": "other",
    "source": ""
  },
  "C57BL_6NJ": {
    "strain": "C57BL/6NJ",
    "id": "C57BL_6NJ",
    "category": "other",
    "source": "https://www.jax.org/strain/005304"
  },
  "C57BR_cdJ": {
    "strain": "C57BR/cdJ",
    "id": "C57BR_cdJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000667"
  },
  "C57L_J": {
    "strain": "C57L/J",
    "id": "C57L_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000668"
  },
  "C58_J": {
    "strain": "C58/J",
    "id": "C58_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000669"
  },
  "CAST_EiJ": {
    "strain": "CAST/EiJ",
    "id": "CAST_EiJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000928"
  },
  "CBA_J": {
    "strain": "CBA/J",
    "id": "CBA_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000656"
  },
  "CE_J": {
    "strain": "CE/J",
    "id": "CE_J",
    "category": "other",
    "source": ""
  },
  "CZECHII_EiJ": {
    "strain": "CZECHII/EiJ",
    "id": "CZECHII_EiJ",
    "category": "other",
    "source": ""
  },
  "DBA_1J": {
    "strain": "DBA/1J",
    "id": "DBA_1J",
    "category": "other",
    "source": "https://www.jax.org/strain/000670"
  },
  "DBA_2J": {
    "strain": "DBA/2J",
    "id": "DBA_2J",
    "category": "other",
    "source": "https://www.jax.org/strain/000671"
  },
  "FVB_NJ": {
    "strain": "FVB/NJ",
    "id": "FVB_NJ",
    "category": "other",
    "source": "https://www.jax.org/strain/001800"
  },
  "I_LnJ": {
    "strain": "I/LnJ",
    "id": "I_LnJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000674"
  },
  "JF1_MsJ": {
    "strain": "JF1/MsJ",
    "id": "JF1_MsJ",
    "category": "other",
    "source": ""
  },
  "KK_HiJ": {
    "strain": "KK/HiJ",
    "id": "KK_HiJ",
    "category": "other",
    "source": ""
  },
  "LEWES_EiJ": {
    "strain": "LEWES/EiJ",
    "id": "LEWES_EiJ",
    "category": "other",
    "source": "https://www.jax.org/strain/002798"
  },
  "LG_J": {
    "strain": "LG/J",
    "id": "LG_J",
    "category": "other",
    "source": ""
  },
  "LP_J": {
    "strain": "LP/J",
    "id": "LP_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000676"
  },
  "MA_MyJ": {
    "strain": "MA/MyJ",
    "id": "MA_MyJ",
    "category": "other",
    "source": ""
  },
  "MOLF_EiJ": {
    "strain": "MOLF/EiJ",
    "id": "MOLF_EiJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000550"
  },
  "NOD_ShiLtJ": {
    "strain": "NOD/ShiLtJ",
    "id": "NOD_ShiLtJ",
    "category": "other",
    "source": "https://www.jax.org/strain/001976"
  },
  "NON_LtJ": {
    "strain": "NON/LtJ",
    "id": "NON_LtJ",
    "category": "other",
    "source": ""
  },
  "NZB_B1NJ": {
    "strain": "NZB/B1NJ",
    "id": "NZB_B1NJ",
    "category": "other",
    "source": ""
  },
  "NZO_HlLtJ": {
    "strain": "NZO/HlLtJ",
    "id": "NZO_HlLtJ",
    "category": "other",
    "source": "https://www.jax.org/strain/002105"
  },
  "NZW_LacJ": {
    "strain": "NZW/LacJ",
    "id": "NZW_LacJ",
    "category": "other",
    "source": "https://www.jax.org/strain/001058"
  },
  "PL_J": {
    "strain": "PL/J",
    "id": "PL_J",
    "category": "other",
    "source": ""
  },
  "PWK_PhJ": {
    "strain": "PWK/PhJ",
    "id": "PWK_PhJ",
    "category": "other",
    "source": "https://www.jax.org/strain/003715"
  },
  "QSi3": {
    "strain": "QSi3",
    "id": "QSi3",
    "category": "other",
    "source": ""
  },
  "QSi5": {
    "strain": "QSi5",
    "id": "QSi5",
    "category": "other",
    "source": ""
  },
  "RF_J": {
    "strain": "RF/J",
    "id": "RF_J",
    "category": "other",
    "source": "https://www.jax.org/strain/000682"
  },
  "RIIIS_J": {
    "strain": "RIIIS/J",
    "id": "RIIIS_J",
    "category": "other",
    "source": ""
  },
  "SEA_GnJ": {
    "strain": "SEA/GnJ",
    "id": "SEA_GnJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000644"
  },
  "SJL_J": {
    "strain": "SJL/J",
    "id": "SJL_J",
    "category": "other",
    "source": ""
  },
  "SM_J": {
    "strain": "SM/J",
    "id": "SM_J",
    "category": "other",
    "source": ""
  },
  "SPRET_EiJ": {
    "strain": "SPRET/EiJ",
    "id": "SPRET_EiJ",
    "category": "other",
    "source": "https://www.jax.org/strain/001146"
  },
  "ST_bJ": {
    "strain": "ST/bJ",
    "id": "ST_bJ",
    "category": "other",
    "source": "https://www.jax.org/strain/000688"
  },
  "SWR_J": {
    "strain": "SWR/J",
    "id": "SWR_J",
    "category": "other",
    "source": ""
  },
  "WSB_EiJ": {
    "strain": "WSB/EiJ",
    "id": "WSB_EiJ",
    "category": "other",
    "source": "https://www.jax.org/strain/001145"
  },
  "ZALENDE_EiJ": {
    "strain": "ZALENDE/EiJ",
    "id": "ZALENDE_EiJ",
    "category": "other",
    "source": "https://www.jax.org/strain/001392"
  }
 }

  if (strain_id) {
    if (strain_id == "all") return id2strain;
    else {
      const ids = strain_id.split(",");
      Object.keys(id2strain).forEach(d => {
        if (!ids.includes(d)) delete id2strain[d];
      });
      return id2strain;
    }
  } else if (strain) { 
    let strain2id  = {};
    Object.keys(id2strain).forEach(d => {
      strain2id[id2strain[d].strain] = id2strain[d];
    });
    if (strain == "all") return strain2id;
    else {
      const strains = strain.split(",");
      Object.keys(strain2id).forEach(d => {
        if (!strains.includes(d)) delete strain2id[d];
      })
      return strain2id;
    }
  }	else {
    return false;
  }
}
