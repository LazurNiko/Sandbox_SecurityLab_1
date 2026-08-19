# Sandbox_SecurityLab_1
An Active Directory lab designed and automated on an isolated virtual infrastructure.

        subgraph VMNET["VMnet2 — Host-only мережа]

            KALI["<b>KALI-ATK</b><br/>Атакер<br/>Kali Linux"]

            VICTIM["<b>VICTIM</b><br/>Ціль<br/>Metasploitable2"]

            SENSOR["<b>SENSOR</b><br/>Моніторинг<br/>Zeek + Suricata"]

            DC01 --- KALI

            KALI --- SENSOR

            SENSOR --- VICTIM

            VICTIM --- DC01

        end

        NIC["Фізичний адаптер хоста<br/>Wi-Fi / Ethernet"]

    end

    NIC -->|"тільки для хоста"| WAN["Домашня мережа / Інтернет"]

    style DC01 fill:#E6F1FB,stroke:#185FA5,color:#0C447C

    style KALI fill:#FAECE7,stroke:#993C1D,color:#712B13

    style VICTIM fill:#FAEEDA,stroke:#854F0B,color:#633806

    style SENSOR fill:#E1F5EE,stroke:#0F6E56,color:#085041

    style NIC fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A

    style WAN fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A

    style VMNET fill:none,stroke:#888780,stroke-dasharray: 4 4

    style HOST fill:none,stroke:#5F5E5A
