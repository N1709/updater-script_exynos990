# updater-script_exynos990

Updater-script and device files for flashing a ROM from a custom recovery.

The flash flow is generic and can be used on any phone, but the updater-script has to be configured for each device. This repository ships a ready configuration for **Samsung Exynos 990** (Galaxy S20 series), with one zip, one system and one vendor base for all three phones.

> **Status:** work in progress. Expect changes while testing continues.

## Included configuration

| Device | Codename | Models |
|---|---|---|
| Galaxy S20 | `x1s` | SM-G980F, SM-G981B |
| Galaxy S20+ | `y2s` | SM-G985F, SM-G986B |
| Galaxy S20 Ultra | `z3s` | SM-G988B |

Detection uses `ro.boot.em.model`. Any other model aborts with `E3004`.

## How it works

1. Checks the device model.
2. Rebuilds the dynamic partition table from `system/dynamic_partitions_op_list` and `unsparse_super_empty.img`.
3. Flashes `system`, `vendor` and `product` with `block_image_update` on the mapped logical partitions.
4. Mounts `vendor` read-write and extracts `device/<codename>/vendor` over it.
5. Writes `odm` to its logical partition, then `dtbo`, `boot` and `up_param` to their block devices.
6. Sets metadata on `/vendor/firmware` and unmounts `vendor`.

## Using it on other devices

Copy the repository and adjust these parts of `updater-script`:

- **Device check:** change the `assert` and the `if` blocks. They compare `getprop("ro.boot.em.model")` against model strings, so replace the property (for example `ro.product.device`) and the strings with the ones your device reports.
- **Per-device blocks:** keep one block per model or codename. Each block extracts `device/<codename>/vendor` and flashes `odm`, `dtbo`, `boot` and `up_param`. Rename the codename in every path and remove the images your device does not have (`up_param` is Samsung specific).
- **Partitions:** keep, add or remove `block_image_update` lines to match the partitions in your package, and update the dat file names.
- **Block device paths:** `boot`, `dtbo` and `up_param` are written to `/dev/block/by-name/<name>`. Change them if your device uses another path or an A/B slot suffix.
- **Dynamic partitions:** edit `system/dynamic_partitions_op_list` with your group name, partition list and sizes. On a device without dynamic partitions, remove `update_dynamic_partitions` and replace `map_partition("<name>")` with `/dev/block/by-name/<NAME>`.
- **Vendor overlay:** if you do not need to copy files over vendor, remove the `mount`, `package_extract_dir` and `set_metadata_recursive` lines for vendor.
- **Folder layout:** the paths inside the script must match the zip structure below.

## Zip structure

```
zip root/
├── META-INF/com/google/android/
│   ├── update-binary
│   └── updater-script
├── system/
│   ├── dynamic_partitions_op_list
│   ├── unsparse_super_empty.img
│   ├── system.new.dat.br
│   ├── system.patch.dat
│   ├── system.transfer.list
│   ├── product.new.dat.br
│   ├── product.patch.dat
│   └── product.transfer.list
├── vendor/
│   ├── vendor.new.dat.br
│   ├── vendor.patch.dat
│   └── vendor.transfer.list
└── device/
    ├── x1s/
    │   ├── system/    boot.img, dtbo.img, odm.img, up_param.bin
    │   └── vendor/    etc, firmware, lib, lib64
    ├── y2s/
    │   ├── system/
    │   └── vendor/
    └── z3s/
        ├── system/
        └── vendor/
```

`system/` and `vendor/` at the root hold the partition images in sparse-dat format. `device/<codename>/system` holds the per-device images, and `device/<codename>/vendor` holds raw files that are copied over the mounted vendor partition.

## Repository structure

This repository only tracks the small files. The large dat images are built locally and are not committed.

```
updater-script_exynos990/
├── META-INF/com/google/android/
│   ├── update-binary
│   └── updater-script
├── device/
│   ├── x1s/
│   ├── y2s/
│   └── z3s/
├── system/
│   ├── dynamic_partitions_op_list
│   └── unsparse_super_empty.img
├── .gitignore
└── README.md
```

## Requirements

- Recovery with dynamic partitions support (`update_dynamic_partitions` and `map_partition`).
- Vendor image in **ext4** with the `shared_blocks` feature removed, otherwise the read-write mount fails. Check and fix with:

```
dumpe2fs -h vendor.img | grep -i features
e2fsck -f -E unshare_blocks vendor.img
```

- `dynamic_partitions_op_list` must declare `system`, `vendor`, `product` and `odm`, and the total size must fit the group and the `super` partition.

## Building the zip

Convert each image to dat and compress it with brotli:

```
img2sdat.py vendor.img -o out -v 4 -p vendor
brotli -j -q 6 out/vendor.new.dat
```

Repeat for `system` and `product`, then copy the results to `system/` and `vendor/` in the zip root. Build the zip from inside the root folder so `META-INF` sits at the top level:

```
zip -r9 ../rom.zip META-INF device system vendor
```

The product size for the op_list can be calculated from its transfer list:

```
echo $(( $(sed -n 2p system/product.transfer.list) * 4096 ))
```

## Error codes

| Code | Meaning |
|---|---|
| `E3004` | Device is not supported |
| `E1001` | Failed to update system image |
| `E2001` | Failed to update vendor image |
| `E3001` | Failed to update product image |
| `E4001` | Failed to mount vendor |

If the install fails, pull the log from recovery and check the last lines:

```
adb pull /tmp/recovery.log
```

## Warning

This script writes to `boot`, `dtbo`, `up_param` and the `super` partition. Keep stock firmware and Odin ready to restore the device. You flash this at your own risk.

## Credits

- Structure reference from the Exynos 8895 updater-script by @nnhglong - telegram.
