# Catalyst Center SWIM upgrade with port maintenance (Ansible)

Ansible version of the Bruno collection: distribute the image, shut selected ports, activate (reload),
wait for the target version, re-enable the ports. Everything goes through the Catalyst Center API;
nothing connects to the switches directly.

Built on `cisco.catalystcenter`, the successor of the deprecated `cisco.dnac` collection.

## Setup

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml -p ./collections
export CATC_USERNAME=apiuser CATC_PASSWORD='...'
```

Edit `group_vars/all/catalystcenter.yml` (host, image, target version), `inventory.yml` (switches) and
`host_vars/<switch>.yml` (`ports_to_cycle`). Set `catc_verify: false` only for a lab without a trusted certificate.

`catc_version` stays at `2.3.7.9` for a 2.3.7.11 cluster: it is the closest version the SDK knows, and it makes
`swim_workflow_manager` use the pre-3.1.3 SWIM API that 2.3.7.x exposes.

## Run

```sh
# 1. Copy the image to flash ahead of the window (no impact)
ansible-playbook swim_upgrade.yml --tags distribute --limit lab-sw01

# 2. The window: snapshot + shut ports, activate, wait for version, re-enable ports
ansible-playbook swim_upgrade.yml --tags upgrade --limit lab-sw01 -e allow_upgrade=true

# Recovery if ports stayed down (uses snapshots/<host>.json)
ansible-playbook restore_ports.yml --limit lab-sw01
```

Without tags, the playbook runs distribution and the window in one go (still gated by `allow_upgrade`).
Hosts are processed one at a time (`serial: 1`).

## Behaviour

- Port state is saved to `snapshots/<host>.json` before any change. Only ports that were admin UP are shut,
  and only those are re-enabled. After a successful restore the file is archived as `<host>.restored-<timestamp>.json`.
- A re-run keeps an existing snapshot for the same device, so ports already shut by a failed run are not
  recorded as "originally down".
- Interface ids are resolved again by name after the reload.
- Each port change waits for its Catalyst Center task, then the playbook resyncs the device and polls the
  inventory until the ports show the expected admin state.
- If anything fails inside the window, the rescue path tries to re-enable the ports
  (`restore_ports_on_failure: true`) and then fails the host.
- The version check ignores leading zeros (17.12.04 matches 17.12.4).

## Not covered

- Whether the shutdown is saved to startup-config before the reload; check this on a lab switch first.
- Image import and golden tagging (both are available in `swim_workflow_manager` if you want them here).
# catalyst-center-swim-ansible
