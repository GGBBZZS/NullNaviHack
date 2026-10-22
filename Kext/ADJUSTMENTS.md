# RX 7600 (Navi 33 dGPU) — Ajustes partir do base Phoenix (GFX1103 APU)

Fonte primária de valores: `../amdgpu log` (76 linhas, Linux real, 0x1002:0x7480 0x1458:0x242B — Gigabyte RX 7600 Gaming OC).
Referência de plano: `Cobalt64-main/src/navi33-bringup/src/Navi33HardwarePlan.h`.

Escrever em **português**. Não fabricar valores de registradores sem o log ou dump real.

---

## 1. Identidade do kext (Info.plist / driver matching)

| Campo          | Valor RX 7600            | Base Phoenix (780M) |
|----------------|--------------------------|---------------------|
| `IOPCIPrimaryMatch`   | `0x1002 0748 0000` (deviceId **0x7480**) | `0x1002 13BF 0000` |
| Vendor            | `0x1002`                | `0x1002`           |
| Subsystem (log)   | Vendor `0x1458` / Device `0x242B` (Gigabyte) | — |
| Class / GFX       | GFX1102, Graphics Core 11 (GC **11.0.2**) | GFX1103, GC 11.0.3 |
| Tipo              | **dGPU** (PCIe dedicada) — NÃO APU | APU (iGPU integrada à CPU) |
| User client       | trocar `'N48N'` → **`'N33N'`** (ou nome dedicado GFX1102) | `'N48N'` |

## 2. Versões de bloco IP (do log amdgpu — Navi 33 real)

Log imprime as **classes** de módulo; alinhar cada camada com a revisão exata
do Navi 33 (ver `Navi33HardwarePlan.h` e histórico do amdgpu).

| Camada IP         | Classe no log            | Revisão Navi 33 a usar      |
|-------------------|--------------------------|-----------------------------|
| SOC / common      | `soc21_common v1_0_0`    | soc21 dGPU                  |
| GMC               | `gmc_v11_0_0`            | GMC 11 (GFX1102)            |
| IH                | `ih_v6_0_0`              | IH 6.0.x                    |
| PSP               | `psp_v13_0_0`            | **psp_v13_0_7** (Navi33)    |
| SMU               | `smu_v13_0_0`            | **smu_v13_0_7** (Navi33)    |
| Display           | `dce_v1_0_0 dm`          | **DCN 3.2.1** (≠ Phoenix 3.1.5 APU) |
| GFX               | `gfx_v11_0_0`            | GFX11 / GC 11.0.2           |
| SDMA              | `sdma_v6_0_0`            | **sdma_v6_0_2**             |
| VCN               | `vcn_v4_0_0`             | VCN 4.0 (log trata instance 0) |
| JPEG              | `jpeg_v4_0_0`            | JPEG 4.0                    |
| MES               | `mes_v11_0_0`            | MES 11.0                    |
| UMC               | (não aparece no log)     | desconhecido — dump Phoenix tem UMC 11.5.0; **não assumir** |

Todos os valores acima correspondem 100% ao log real.

### Cambios de versão Phoenix → RX 7600

- **Garantir:** tirar todas as referências `psp_v13_0_4 / smu` vlast `APU` e usar **13.0.7**.
- **Garantir:** display **DCN 3.2.1** ≠ **DCN 3.1.5** (Phoenix).
- **Garantir:** SDMA **6.0.2** (Phoenix usa variante APU de SDMA 6.0.x).

## 3. Geometria VRAM / GART (valores reais do log)

| Item                  | Valor RX 7600 (RX 7600 Gaming OC, 8 GB GDDR6 128-bit) |
|-----------------------|--------------------------------------------------------|
| VRAM real (`vram`)    | **`8176M 0x8000000000-0x81FEFFFFFF (8176M used)`** — usados reais **8176** MiB, **não** 8192 |
| BAR0 (reCAR/reBAR)    | **8192M** (`8192M flops …`) —>` assignedBar0Bytes` ≠ `realBytes` (usar `PhysicalInventory` de `Navi33Gmc11VramGeometry.h`; `realBytes == 8176 MiB`) |
| MMIO BAR0             | base `0x60000000`, tamanho **1048576** (1 MB) — `mmio [0x60000000-0x600FFFFF]` |
| GART                  | **512M** reservado em `0x00007FFF00000000-0x00007FFF1FFFFFFF`; PCIE GART table @ **0x81FEB00000** |
| GART page             | 131072 páginas (`131072 pages in GART`), PCIEg = **4 levels**, block/fragment `9/9-bit` |
| VM                    | 262144 GiB espaço, 4 levels, LGTM `x86_64 block` |

## 4. Flag `kGbAddrConfig` (⚠️ NÃO CONFIRMADO PARA dGPU)

- `kGbAddrConfig = 0x00000242` em `PhxBringup.cpp` L51/L1339 vem do dump do **Firebat U6C APU**.
- O log amdgpu NÃO imprime o valor de `GB_ADDR_CONFIG` da RX 7600.
- **Proibido usar 0x00000242** no kext GFX1102 até confirmação real.
- Ação: obter via dump IP/MMIO (ex.: `capture-linux-ipd.sh`, IP GFX1102 no Linux) ou dump de registradores.
- Idem para **MTYPE** (`Navi33Gmc11Gart.h` PTE MTYPE bits 50:48, `kMtypeUC=3<<48`) — confirmar valor dGPU.

## 5. Display (⚠️ Reset de caminho)

- **Descartar** display APU de Phoenix (DCN 3.1.5 / DMUB APU phx).
- pageTitle **DCN 3.2.1** nos paths RDNA3Framebuffer/na variante RDNA3-DGPU.
- **DMUB** `0x07003300` via PSP (log: `DMUB firmware loaded`) — exige PSP 13.0.7.
- `Display Core v3.2.384` driver em **DCN 3.2.1**; **DP-HDMI FRL PCON** suportado no path de display (log).
- VBIOS `ATOMBIOS: 113-R76GO-F91 build 00175427 ver 022.003.001.037.000001 (2026/01/18)` via **VFCT** (Linux).
  - macOS não tem tabela VFCT.ACPI → embutir a imagem VBIOS como blob no kext ou firmware bundle (não extrair de macOS).
- `reg_wait timeout 1us * 150000 tries - optc32_disable_crtc` —****benigno e recorrente, **não tratar como falha** no条件的 bringup.

## 6. Módulos desligados / opcionais

- **VCN 4.0 / JPEG 4.0 — MANTER DESLIGADO** (guarantee do kext não tranca); sem decode de vídeo no kext GFX1102 inicial.
- **MES 11.0 — OPCIONAL** (fase posterior); log trata instável (masks: `vmid_mask_mmhub/gfxhub 0x0000ff00`, `gfx_hqd_mask 0x2`, `compute_hqd_mask 0xc`, `sdma_hqd_mask 0xfc`).
- **TMZ non suportado** (log: `amdgpu: TMZ is not supported`) — não habilitar.
- **CP RS64 enable** (o Samsungo o log): microcode RS64 (MEC/ME RS64) **é obrigatório** — incluir em fetch firmware.
- **CRAT virtual** (`virtual CRAT table`) — irrelevante para macOS; pular.

## 7. PSP

- PSP **13.0.7** (Fixar revision; a classe do log é `psp_v13_0_0`).
  - `psp_gfx_rb_frame` 64B ring (5 campos) — conforme Linux `amdgpu_psp.c:3485` (~linha); manter formato Phoenix.
  - **TMR** (TMR size no log): `0x1300000 @ 0x81fc000000` (TMR @ 0x81fc000000, size 0x1300000) -> Use exato no kext.
  - **RAS / RAP / SECUREDISPLAY TA——** opcionais e absentes (os 3: "Padding pattern is not initialized", "RAS/SECUREDISPLAY TA"): pular.
  - Firmware blobs necessários: `psp_13_0_7_*.bin`, `gc_11_0_2.pfw` (PFP/CE/ME RS64), `sdma_6_0_2.bin`, `dmub dcn_3_2_1` (`dmub/dcn_3_2_1/`), `smu_13_0_7.bin` (`smu_13_0_7_0x00526100.bin`) — usar `tools/fetch_test_firmware.sh` como base.
- **SMU 13.0.7**: fw version `0x00526100` (82.97.0), driver-if `0x35`, smu-fw-if `0x40` (SMU ok). Usm16 options (pode pular poder/HW no kext inicial).

## 8. `MES` Masks/HQD (para referência futura)

- `vmid_mask_mmhub/gfxhub:0x0000ff00` — note Addshift Mask/own msk (garantir no código).
- `gfx_hqd_mask 0x2`, `compute_hqd_mask 0xc`, `sdma_hqd_mask 0xfc` (sdma engine list 0..7 split) — cachar no plano GFX1102.

## 9. Caminho de integração

- Base do fork: nullarc **Phoenix** (--arvores `src/` + `tools/` copiadas para `../CF/`).
  Copiar tooling para `CF/tools/` e:
  1. **Clonar** `build_kext.py` → **`build_r7600_kext.py`** (trocar identidade, dGPU);
  2. Trocar constantes APU→dGPU em `CF/` (deviceId/GC 11.0.2/DCN 3.2.1, remove display APU);
  3. Alinhar gates Cobalt64 (`Navi33Gmc11Vram*`, `Navi33NativeABI.h`, `Navi33StageOwner.h`) para dGPU.
- A estrutura invariante **96 bytes**[`n33_vram_geometry_status`] e **64 bytes**[`n33_*`] uso estritamente como está (sem esqueleto de mudanças).

Procure o lvma `fetch_test_firmware.sh` para todos os blobs (psp_13_0_7, gc_11_0_2, sdma_6_0_2, dcn_3_2_1, smu_13_0_7).
GFX1102 ip_discovery — `capture-linux-ipd.sh`.
