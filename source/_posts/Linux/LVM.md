---
title: LVM
categories: 
- linux
tags: 
- 未整理
---

## PV操作

### 创建PV
`pvcreate [devices]`

### 查询PV
`pvscan`
`pvdisplay [pv names]`

### PV大小修改
`pvresize`

### 删除PV
`pvremove`

## VG操作

### 创建VG
`vgcreate -s [pe大小，默认4M] [vg name] [pv names]`

### 查询VG
`vgscan`
`vgdisplay [vgnames]`

### 删除VG
`vgremove`

## LV操作

### 创建LV
`lvcreate -L [lv大小] -n [lv name] [vg name]`

### 查询LV
`lvscan`
`lvdisplay [lvnames]`

### PV大小修改
`pvresize`

### 删除LV
`lvremove`