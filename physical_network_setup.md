# Physical Network Setup Guide - Step by Step
==============================================

## 🎯 **Goal**
Set up a secure 3-zone network with proper physical connections for the Network Security Lab.

## 📦 **Equipment Checklist**
Before starting, make sure you have:
- [ ] 5 grml computers (labeled grml1 through grml5)
- [ ] 2-3 Ethernet switches (small boxes with multiple network ports)
- [ ] 8-10 Ethernet cables (patch cables with RJ45 connectors)
- [ ] Access to wall port EV3-2-2 5/9 or EV3-2-2 5/21

---

## 🏗️ **Network Architecture Overview**

```
🌐 INTERNET (141.76.46.0/24)
     ↓
  Wall Port EV3-2-2
     ↓
┌─────────────────┐
│     grml1       │ ← Main Traffic Controller (NAT Gateway)
│  (3 interfaces) │
└─────┬─────┬─────┘
      ↓     ↓
   DMZ     INTERNAL
   Zone     Zone
     ↓       ↓
grml2,grml3   grml4,grml5
(DNS,Web)     (Clients)
```

---

## 🔌 **Step 1: Connect grml1 (NAT Gateway) - The Central Hub**

**grml1 is your network's "traffic director" and needs 3 separate connections:**

### 🌐 Connection A: Internet Access (eth0)
```
Internet → Wall Port → Cable → grml1 eth0
```
**Instructions:**
1. Locate wall port **EV2.2.2 - 5/14**
2. Take 1 Ethernet cable
3. Plug one end into the **wall port**
4. Plug other end into **grml1's eth0** (usually first/leftmost Ethernet port)
5. **Result:** grml1 now has Internet access (141.76.46.220)

### 🔒 Connection B: DMZ Network (eth1)
```
grml1 eth1 → Cable → Switch A
```
**Instructions:**
1. Take 1 Ethernet cable
2. Plug one end into **grml1's eth1** (second Ethernet port)
3. Plug other end into **Switch A** (any port - we'll call this "DMZ Switch")
4. **Label Switch A** as "DMZ" to avoid confusion
5. **Result:** grml1 can now control DMZ traffic (192.168.2.1)

### 🏠 Connection C: Internal Network (eth2)
```
grml1 eth2 → Cable → Switch B
```
**Instructions:**
1. Take 1 Ethernet cable
2. Plug one end into **grml1's eth2** (third Ethernet port)
3. Plug other end into **Switch B** (any port - we'll call this "Internal Switch")
4. **Label Switch B** as "INTERNAL" to avoid confusion
5. **Result:** grml1 can now control internal traffic (192.168.1.1)

---

## 🔒 **Step 2: Connect DMZ Zone (Semi-Public Area)**

**DMZ = Demilitarized Zone - for DNS and web servers**

### Connect grml2 (DNS Server) and grml3 (Web Server)
```
Switch A (DMZ) connections:
├─ grml1 eth1 (already connected)
├─ grml2 (DNS Server)
└─ grml3 (Web Server)
```
**Instructions:**
1. **grml2 (DNS):** Cable from Switch A → grml2 eth0
2. **grml3 (Web):** Cable from Switch A → grml3 eth0

**DMZ Zone Complete:**
- Switch A has 3 cables: grml1 eth1 + grml2 + grml3
- Both servers are isolated from client machines
- DNS and Web servers can communicate within DMZ

---

## 🏠 **Step 3: Connect Internal Zone (Private Area)**

**Internal = Your secure private network**

### Connect Client Workstations
```
Switch B (Internal) connections:
├─ grml1 eth2 (already connected)
├─ grml4 (Client 1)
└─ grml5 (Client 2)
```

**Instructions:**
1. **grml4 (Client 1):** Cable from Switch B → grml4 eth0
2. **grml5 (Client 2):** Cable from Switch B → grml5 eth0

**Internal Zone Complete:**
- Switch B has 3 cables total
- Client machines can talk to each other
- Clients reach DNS/Web through grml1 (controlled access)

---

## 📊 **Final Physical Layout**

```
                    🌐 INTERNET
                         ↓
                   Wall Port EV3-2-2
                         ↓
                    ┌─────────┐
                    │  grml1  │
                    │   NAT   │
                    └─┬─┬─┬───┘
                    eth0│ │eth2
                        │ │
                     eth1│ └─────┐
                         │       │
               ┌─────────▼─┐   ┌─▼─────────────────┐
               │ Switch A  │   │    Switch B       │
               │   (DMZ)   │   │  (INTERNAL)       │
               └─────┬─────┘   └─┬─┬─┬─────────────┘
                     │           │ │ │
                ┌────▼────┐     │ │ └───grml5 (CLT2)
                │  grml3  │     │ └─────grml4 (CLT1)
                │  (WEB)  │     └───────grml2 (DNS)
                └─────────┘
```

---

## 🔍 **Verification Checklist**

After connecting all cables, verify:

### Visual Check
- [ ] All Ethernet ports show **green/orange LED lights**
- [ ] **7 total cables** used (1 to wall + 6 between machines/switches)
- [ ] **grml1 has 3 cables** (eth0, eth1, eth2)
- [ ] **Switch A has 3 cables** (grml1 + grml2 + grml3)
- [ ] **Switch B has 3 cables** (grml1 + 2 client machines)

### Physical Connection Map
```
Cable 1: Wall Port EV2.2.2 - 5/14 → grml1 eth0
Cable 2: grml1 eth1 → Switch A
Cable 3: grml1 eth2 → Switch B
Cable 4: Switch A → grml2 eth0 (DNS)
Cable 5: Switch A → grml3 eth0 (Web)
Cable 6: Switch B → grml4 eth0 (Client)
Cable 7: Switch B → grml5 eth0 (Client)
```

### Zone Verification
- [ ] **DMZ Zone:** grml2 (DNS) and grml3 (Web) connected
- [ ] **Internal Zone:** Only grml4, grml5 (clients) connected
- [ ] **No cross-connections:** DNS/Web servers NOT on internal switch

---

## 🚨 **Common Mistakes & Troubleshooting**

### Mistake 1: Wrong grml1 Interfaces
**Problem:** Mixed up eth0, eth1, eth2 on grml1
**Solution:** 
- eth0 = Internet (to wall)
- eth1 = DMZ (to Switch A)
- eth2 = Internal (to Switch B)

### Mistake 2: Mixed Network Zones
**Problem:** grml3 connected to internal switch
**Solution:** grml3 MUST be on Switch A (DMZ only)

### Mistake 3: No LED Lights
**Problem:** Ethernet port shows no lights
**Solution:** 
- Check cable is fully inserted (should "click")
- Try different cable
- Try different switch port

### Mistake 4: Switch Confusion
**Problem:** Don't know which switch is which
**Solution:** 
- Label switches clearly: "DMZ" and "INTERNAL"
- Count connections: DMZ=2 cables, Internal=4 cables

---

## 🎯 **What Happens Next**

After physical setup is complete:
1. **Configure IP addresses** on each machine
2. **Set up iptables rules** on grml1 for security
3. **Install services** (DNS on grml2, Web on grml3)
4. **Test connectivity** between zones

**Remember:** Physical connections create the foundation, but software configuration controls the security!

---

## 📞 **Quick Reference**

### IP Address Plan
- **grml1:** 141.76.46.220 (eth0), 192.168.2.1 (eth1), 192.168.1.1 (eth2)
- **grml2:** 192.168.2.2 (DNS Server - DMZ)
- **grml3:** 192.168.2.10 (Web Server - DMZ)
- **grml4:** 192.168.1.4 (Client 1)
- **grml5:** 192.168.1.5 (Client 2)

### Network Purposes
- **External (141.76.46.x):** Internet connectivity
- **DMZ (192.168.2.x):** Public web services (isolated)
- **Internal (192.168.1.x):** Private resources (protected)

**✅ Physical setup complete! Ready for software configuration.** 