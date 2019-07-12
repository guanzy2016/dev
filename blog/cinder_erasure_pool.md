# openstack 使用 ceph erasure pool

[ec pool 简介](https://www.cnblogs.com/damizhou/p/5737902.html)

k + m 原理：

k 份原始数据，m 份校验，通过 k 份原始数据，根据算法生成新的 k + m 份数据，可允许任意 m 份数据丢失，都会根据算法找回，所以使用纠删码，要保证 ceph 集群有 k + m 个节点或是 osd（可在 erasure-code-profile 中设置）。

## 设置 profile

```
ceph osd erasure-code-profile set ec_profile crush-failure-domain=osd \
crush-root=default \
k=2 \
m=1 \
plugin=isa
```

- crush-failure-domain=osd 数据份设置为 osd 级别，可设置为 host，需要集群有 k + m 可节点

- plugin=isa 为计算算法

## 创建 ec pool

```
ceph osd pool create ec_pool 12 12 erasure ec_profile
```

## 支持 RBD 和 CephFS

```
ceph osd pool set ec_pool allow_ec_overwrites true
```
默认 ec pool 只能在 RGW 上使用。

## 应用到 metadata pool

```
ceph osd pool application enable volumes ec_pool
```

cinder 使用的 volumes pool 作为 metadata-pool，ec_pool 作为 data-pool

## 测试

```
rbd create --size 1G --data-pool ec_pool volumes/ec_vol
```

```
rbd info volumes/ec_vol
rbd image 'ec_vol':
	size 1GiB in 256 objects
	order 22 (4MiB objects)
	data_pool: ec_pool
	block_name_prefix: rbd_data.2.55f5a6b8b4567
	format: 2
	features: layering, striping, exclusive-lock, object-map, fast-diff, deep-flatten, data-pool
	flags:
	create_timestamp: Fri Jul 12 15:09:24 2019
	stripe unit: 1MiB
	stripe count: 32
```

## 配置 ceph.conf

```
cp /etc/ceph/ceph.conf /etc/ceph/ceph_ec.conf

vi /etc/ceph/ceph_ec.conf

[client.admin]
rbd_default_data_pool = ec_pool
```

## 配置 cinder.conf

```
[rbd-ec]
rbd_ceph_conf = /etc/ceph/ceph_ec.conf
rbd_user = admin
backend_host = ceph-dev
rbd_pool = volumes
volume_backend_name = rbd-ec
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid = 27a2e46a-c070-4f18-acd8-47d9bbd42407
```

重启 cinder-volume 服务即可。

注：这里测试用 admin 用户创建的 ec\_pool，所以 cinder 中配置的 rbd\_user 为 admin，如果是 cinder，需要为 cinder 用户增加对 ec\_pool 的访问权限。

