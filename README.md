
# Hierarchical Threshold Signature Scheme
[![Apache licensed][1]][2] [![Go Report Card][3]][4]

[1]: https://img.shields.io/badge/License-Apache%202.0-blue.svg
[2]: LICENSE
[3]: https://goreportcard.com/badge/github.com/getamis/alice
[4]: https://goreportcard.com/report/github.com/getamis/alice

## Introduction:

This is an implementation for multi-party TSS library with Hierarchical shares(abrev. HTSS). We quote one paragraph of  [Tassa's paper](https://www.openu.ac.il/lists/mediaserver_documents/personalsites/tamirtassa/hss_conf.pdf) to explain the difference between traditional threshold signature scheme and HTSS:


 **There are many real-life examples of threshold secret sharing.**
 **Typical examples include sharing a key to the central vault in a bank, the triggering mechanism for nuclear weapons, or key escrow.**
 **We would like to consider here a special kind of generalized secret sharing scenarios that is a natural extension of threshold secret sharing.**
 **In all of the above mentioned examples, it is natural to expect that the participants are not equal in their privileges or authorities.**
 **For example, in the bank scenario, the shares of the vault key may be distributed among bank employees,** 
 **some of whom are tellers and some are department managers.**
 **The bank policy could require the presence of, say, 3 employees in opening the vault,**
 **but at least one of them must be a department manager. Or in key escrow, the dealer might demand that some escrow agents (say, family members) must be involved in any** 
 **emergency access to his private files. Such settings call for special methods of secret sharing.**

> One of references of HTSS is [A Hierarchical Threshold Signature](https://medium.com/getamis/a-hierarchical-threshold-signature-830683b87873) or See [Example](#Example).



## Table of Contents:

*	[Introduction](#Introduction)
*	[Implementations](#implementation)
	*	[DKG](#DKG)
    *	[Signer](#Signer)
		*	[GG18](#GG18)
		*	[CCLST](#CCLST)
    *	[Reshare](#Reshare)
*	[Usage](#usage)
    *	[Peer](#peerusage)
    *   [Listener](#listenerusage)
    *	[DKG](#DKGusage)
    *	[Signer](#signerusage)
    *	[Reshare](#reshareusage)
*	[Examples](#Example)
    *	[Standard threshold signature](#lagrangecase)
    	*	[DKG](#DKGLagrangeCase)
		*	[Signer](#SigningLagrangeCase)
	*	[Hierarchical threshold signature](#Hierarchicalthresholdsignature)
		*	[DKG](#DKGH)
		*	[Signer](#SignerH)
*	[Benchmarks](#benchmark)
	*	[GG18](#gg18benchmark)
	*	[CCLST](#cclstbenchmark)
*	[Appendix](#appendix)
	*	[Security levels of two homomorphic schemes](#securitylevel)
	*	[Useful Cryptography Libraries](#usefulLibrary)	
*	[References](#reference)
*	[Other Libraries](Libraries)


<h2 id="implementation">Implementations:</h2>

Like the classical TSS, HTSS also contains three protocols:

1. DKG: Distributed key generation for creating secret shares without any dealer.
2. Signer: Signing for using the secret shares to generate a signature.
3. Reshare: Refresh the secret share without changing the public key.

**Remark:**
1. Comparing to TSS, each share in HTSS is  generated by DKG having difference levels (or called rank). The level 0 is the highest.
2. If all levels of shares are zero, then HTSS reduces to the classical TSS. (i.e. In this case, Birkhoff interpolation reduces to Lagrange Interpolation).
3. After perform the progress of DKG, each participant will get (x-coordinate, share, rank). Assume that the threshold is 3. Therefore, any three shares (x-coord, rank): (x1, n1), (x2, n2), (x3, n3)  with n1 <= n2 <= n3  can recover the secret or sign if and only if 
n_i <= i-1 for all 1 <= i <= 3. In general, assuming the threshold is t, any t shares (xi,ni) with ni <= nj for all i < j can recover the secret or sign if and only if n_i <= i-1 for all 1 <= i <= t. In brief, the authorized set with order t of all participants is n_i <= i-1 for all 1 <= i <= t.

**Example:**

Let threshold = 3, and participants = 5. Assume the corresponding rank of each shareholder are 0, 0, 1, 1, 2. Then authorized sets in this setting are 
1. 0, 0, 1
2. 0, 1, 1
3. 0, 1, 2

The other combinations of shares can not recover the secret (eg. 1, 1, 2).


<h3 id="DKG">DKG:</h3>

We implement a modified version of DKG in [Fast Multiparty Threshold ECDSA with Fast Trustless Setup](https://eprint.iacr.org/2019/114.pdf).
We point out the different parts:
* Replace Lagrange interpolation with [Birkhoff interpolation](https://en.wikipedia.org/wiki/Birkhoff_interpolation) and generate own x-coordinate respectively.
* We do not generate a private key and the corresponding public key of homomorphic encryptions (i.e. Paillier cryptosystem or CL Scheme) in the key-generation. Move it to the beginning of Signer.

<h3 id="Signer">Signer:</h3>

Our implementation involves two algorithms: [Fast Multiparty Threshold ECDSA with Fast Trustless Setup](https://eprint.iacr.org/2019/114.pdf) and [Bandwidth-efficient threshold EC-DSA](https://eprint.iacr.org/2020/084).
The main difference of two schemes is applying different homomorphic encryptions. One is Paillier cryptosystem and another is CL scheme.
We point out the difference between our implementation and their versions.

<h4 id="GG18">GG18:</h4>

* Replace Lagrange interpolation with [Birkhoff interpolation](https://en.wikipedia.org/wiki/Birkhoff_interpolation).
* In the beginning of signer, we generate a key-pair of the homomorphic encryption. 

**Remark:**
Our version is the algorithm of GG18 without doing range proofs in MtA(cf. [Section 3, GG18](https://eprint.iacr.org/2019/114.pdf)). 


<h4 id="CCLST">CCLST:</h4>

* Replace Lagrange interpolation with [Birkhoff interpolation](https://en.wikipedia.org/wiki/Birkhoff_interpolation).
* In the beginning of signer, we generate a key-pair of the homomorphic encryption.


<h3 id="Reshare">Reshare:</h3>

It is the standard algorithm replacing Lagrange interpolation with [Birkhoff interpolation](https://en.wikipedia.org/wiki/Birkhoff_interpolation).

<h2 id="usage">Usage:</h2>

<h3 id="peerusage">Peer:</h3>
We create an interface named `PeerManager`. It defines three methods to be implemented.

1. `NumPeers`: Return the number of peers.
2. `SelfID`: Return the self ID.
3. `MustSend`: Send message to the specific peer.

Before you try to go through a multi-party algorithm, you should create a peer manager instance first. Here is an example for DKG.
```
type dkgPeerManager struct {
	id       string
	numPeers uint32
	dkgs     map[string]*dkg.DKG
}

func newDKGPeerManager(id string, numPeers int, dkgs map[string]*dkg.DKG) *dkgPeerManager {
	return &dkgPeerManager{
		id:       id,
		numPeers: uint32(numPeers),
		dkgs:     dkgs,
	}
}

func (p *dkgPeerManager) NumPeers() uint32 {
	return p.numPeers
}

func (p *dkgPeerManager) SelfID() string {
	return p.id
}

func (p *dkgPeerManager) MustSend(id string, message proto.Message) {
	d := p.dkgs[id]
	msg := message.(types.Message)
	d.AddMessage(msg)
}
```

<h3 id="listenerusage">Listener:</h3>
We create an interface named `StateChangedListener`. It defines one method to be implemented.

1. `OnStateChanged`: Handle state changed.

```
type listener struct{}

func newListener() *listener {
	return &listener{}
}

func (l *listener) OnStateChanged(oldState types.MainState, newState types.MainState) {
	// Handle state changed.
}
```

<h3 id="DKGusage">DKG:</h3>

Distributed key generation (DKG) is a process that multiple parties participate in the calculation of a shared public key and private key.
To create a new DKG instance, there are some inputs caller must provide in our implementation.

* **curve**: an elliptic curve (e.g S256)
* **dkgPeerManager**: a peer manager for DKG
* **threshold**: minimum number of participants required to sign
* **rank**: the rank of this participant
* **listener**: a function to monitor the state change

```
myDKG, err := dkg.NewDKG(curve, dkgPeerManager, threshold, rank, listener)
if err != nil {
    // handle error
}
myDKG.Start()
// send out peer message...
myDKG.Stop()
dkgResult, err := myDKG.GetResult()
if err != nil {
    // handle error
}
```

After DKG, all the participants would get the same public key and all the x-coordinates and ranks. Each participant would also get their own share.
<h3 id="signerusage">Signer:</h3>

A (t,n)-threshold signature is a digital signature scheme that any t or more signers of a group of n signers could generate a valid signature. Here, we support two encryption algorithms for signing: Paillier, and CL. Caller must specify which encryption to be used. The security level of two homomorphic encryptions can be found in [Appendix](#appendix).

```
// 1. Paillier
homo, err := paillier.NewPaillier(2048)
if err != nil {
    // handle error
}
// 2. CL
safeParameter := 1348
distributionDistance := uint(40)
homo, err := cl.NewCL(big.NewInt(1024), 40, c.Params().N, safeParameter, distributionDistance)
if err != nil {
    // handle error
}
```

To start signing, you should provide some inputs for creating a new Signer instance.

* **signerPeerManager**: a peer manager for signer
* **publicKey**: the public key generated from DKG
* **homo**: a homomorphic encryption (Paillier of CL)
* **share**: the private share from DKG
* **selfBk**: the Birkhoff parameter of this participant
* **peerBks**: Birkhoff parameters of other participants
* **msg**: a message to be signed
* **listener**: a function to monitor the state change

Note that, `threshold` signers are required to participate in signing. Therefore, the length of `peerBks` must not be less than `threshold-1`.

```
mySigner, err = signer.NewSigner(signerPeerManager, publicKey, homo, share, selfBk, peerBks, msg, listener)
if err != nil {
    // handle error
}
mySigner.Start()
// send out public key message...
mySigner.Stop()
signerResult, err := mySigner.GetResult()
if err != nil {
    // handle error
}
```

After signing, all the participants should get the same signature.

<h3 id="reshareusage">Reshare:</h3>

Refreshing share (reshare) computes new random shares for the same original secret key. Before resharing, here is also some inputs you need to prepare.

* **resharePeerManager**: a peer manager for reshare
* **threshold**: minimum number of participants required to sign
* **publicKey**: the public key generated from DKG
* **share**: the private share from DKG
* **bks**: Birkhoff parameters from all participants
* **listener**: a function to monitor the state change

Note that, reshare process requires all peers to participate.

```
myReshare, err = reshare.NewReshare(resharePeerManager, threshold, publicKey, share, bks, listener)
if err != nil {
    // handle error
}
myReshare.Start()
// send out commit message...
myReshare.Stop()
signerResult, err := myReshare.GetResult()
if err != nil {
    // handle error
}
```
After resharing, all the participants should get their new shares. For more usage, please check `tss/integration/tee_test.go`.

<h2 id="Example">Examples:</h2>

<h3 id="lagrangecase">Standard threshold signature:</h3>

There are three people co-founding a small store. Many goods come in and out every day. To increase efficiency, they agree that if any two of them confirms a transaction, the transaction should be valid. In this case, they could utilize threshold signature.

In DKG stage, all of them should create a peer manager specifying number of peers to be `2`. And all of them should have the same value of `rank` to be `0`. Then in signing stage, any two of them could generate a valid signature.

* Ranks: (0, 0, 0)
* Threshold: 2


<h4 id="DKGLagrangeCase">DKG:</h4>

```
// S256 for example
curve := btcec.S256()
id := "myID"
peerNum := uint32(2)
threshold := uint32(2)
rank := uint32(0)

// each co-founder
dkgPeerManager := newDKGPeerManager(id, peerNum, dkgs)
myDKG, _ := dkg.NewDKG(curve, dkgPeerManager, threshold, rank, listener)
```

<h4 id="SigningLagrangeCase">Signing:</h4>

```
msg := []byte{1, 2, 3}
peerNum := uint32(1)
homo, _ := paillier.NewPaillier(2048)

// get result from DKG
result, _ := myDKG.GetResult()

// any two of co-founders
signerPeerManager := newSignerPeerManager(id, peerNum, signers)
mySigner, _ := signer.NewSigner(signerPeerManager, result.PublicKey, homo, result.Share, result.Bks[id], peerBks, msg, listener)
```

<h3 id="Hierarchicalthresholdsignature">Hierarchical threshold signature:</h3>
Imagine a department in a company is consisted of three employees and one director. The company stipulate that any transaction should be approved by at least three people and one of the approval must be the director. In this case, there are four participants but their powers might be different.

In DKG stage, all of them should create a peer manager specifying number of peers to be `3`. Three employees should set the `rank` value to be `1` but the director should have the `rank` value to be `0` (smaller the value, higher the rank). In signing stage, any two of employees along with the director could generate a valid signature. If three employees without the director try to sign a message, the signing process will return an error.

* Ranks: (0, 1, 1, 1)
* Threshold: 3

<h4 id="DKGH">DKG:</h4>

```
// S256 for example
curve := btcec.S256()
id := "myID"
peerNum := uint32(3)
threshold := uint32(3)
employeeRank := uint32(1)
directorRank := uint32(0)

// each employee
dkgPeerManager := newDKGPeerManager(id, peerNum, dkgs)
employeeDKG, _ := dkg.NewDKG(curve, dkgPeerManager, threshold, employeeRank, listener)

// the director
directorDKG, _ := dkg.NewDKG(curve, dkgPeerManager, threshold, directorRank, listener)
```

<h4 id="SignerH">Signer:</h4>

```
msg := []byte{1, 2, 3}
peerNum := uint32(3)
homo, _ := paillier.NewPaillier(2048)

// get result from DKG (for employee)
result, _ := employeeDKG.GetResult()
// get result from DKG (for director)
result, _ := directorDKG.GetResult()

// two of employees and the director
signerPeerManager := newSignerPeerManager(id, peerNum, signers)
mySigner, err = signer.NewSigner(signerPeerManager, result.PublicKey, homo, result.Share, result.Bks[id], peerBks, msg, listener)
```

<h2 id="benchmark">Benchmarks:</h2>

Our benchmarks were in local computation and ran on an Intel qualcore-i5 CPU 2.3 GHz and 16GB of RAM.

<h3 id="gg18benchmark">GG18(Paillier case):</h3>

* Threshold: 3
* Three shares with ranks (x, share, rank): (108,4517821,0),(344,35822,1),(756,46,2)
* Public Key: 2089765G
* The bit-length of Paillier public Key: 2048 (ref. [Appexdix](appendix))
* curve: [S256](https://github.com/ethereum/go-ethereum/blob/0bdb21f0cb9fe73a02b61aeedbbae5b57fc337ae/crypto/signature_cgo.go#L85)

(Ran 10 samples)
```
+-----------------+--------------------------+------------------------------+
|  Fastest Time   |            Slowest Time  |                 Average Time |
+-----------------+--------------------------+------------------------------+
|          0.719s |                   1.150s |              0.902s ± 0.125s |
+-----------------+--------------------------+------------------------------+
```

<h3 id="cclstbenchmark">For CCLST:</h3>

* Threshold: 3
* Three shares with ranks (x, share, rank): (108,4517821,0),(344,35822,1),(756,46,2)
* Public Key: 2089765G
* The security level: 1348(i.e. the bit length of fundamental discriminant) (ref. [Appendix](#appendix))
* distribution distance: 40 (cf. [Section 3.1](https://eprint.iacr.org/2020/084))
* the challenge set C: 1024 (cf. [Section 3.1](https://eprint.iacr.org/2020/084))
* r1 challenge bit translation: 40 (cf. [Section 3.1](https://eprint.iacr.org/2020/084))
* curve: [S256](https://github.com/ethereum/go-ethereum/blob/0bdb21f0cb9fe73a02b61aeedbbae5b57fc337ae/crypto/signature_cgo.go#L85)

(Ran 10 samples)
```
+-----------------+--------------------------+------------------------------+
|  Fastest Time   |            Slowest Time  |                 Average Time |
+-----------------+--------------------------+------------------------------+
|          2.452s |                   3.754s |              3.229s ± 0.396s |
+-----------------+--------------------------+------------------------------+
```
<h2 id="appendix">Appendix:</h2>

<h3 id="securitylevel">Security levels of two homomorphic schemes:</h3>

The Table below is referenced by [Improved Efficiency of a Linearly Homomorphic Cryptosystem](https://link.springer.com/chapter/10.1007/978-3-030-16458-4_20).

```
+-----------------+--------------------------+-----------------------------------+
| Security Level  |  RSA modulus (Paillier)  |  fundamental discriminant ΔK (CL) |
+-----------------+--------------------------+-----------------------------------+
|          112    |          2048            |                         1348      |
|          128    |          3072            |                         1828      |
|          192    |          7680            |                         3598      |
|          256    |         15360            |                         5972      |
+-----------------+--------------------------+-----------------------------------+
```

<h3 id="usefulLibrary">Useful Cryptography Libraries:</h3>

1. [Binary quadratic forms for class groups of imaginary quadratic fields](https://github.com/getamis/alice/tree/master/crypto/binaryquadraticform)



<h2 id="reference">References:</h2>

1. [Fast Multiparty Threshold ECDSA with Fast Trustless Setup](https://eprint.iacr.org/2019/114.pdf)
2. [Bandwidth-efficient threshold EC-DSA](https://eprint.iacr.org/2020/084)
3. [Hierarchical Threshold Secret Sharing](https://www.openu.ac.il/lists/mediaserver_documents/personalsites/tamirtassa/hss_conf.pdf)
4. [Dynamic and Verifiable Hierarchical Secret Sharing](https://eprint.iacr.org/2017/724.pdf)
5. [Linearly Homomorphic Encryption from DDH](https://pdfs.semanticscholar.org/fba2/b7806ea103b41e411792a87a18972c2777d2.pdf?_ga=2.188920107.1077232223.1562737567-609154886.1559798768)
6. [Improved Efficiency of a Linearly Homomorphic Cryptosystem](https://link.springer.com/chapter/10.1007/978-3-030-16458-4_20)
7. [A Course in Computational Algebraic Number Theory (Graduate Texts in Mathematics)](https://www.amazon.com/Course-Computational-Algebraic-Graduate-Mathematics/dp/3540556400)
8. [Maxwell Sayles: binary quadratic form](https://github.com/maxwellsayles/)

<h2 id="Libraries">Other Libraries:</h2>

1. [KZen-tss](https://github.com/KZen-networks/multi-party-ecdsa)
2. [Binance-tss](https://github.com/binance-chain/tss-lib)
