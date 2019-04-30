# Anjuna Unseal tool for Hashicorp Vault

<img src="https://docs.anjuna.io/vault-unseal/_static/unseal_logo.png" width="50%">

See documentation on   
https://docs.anjuna.io/vault-unseal  
or  
https://anjuna-security.github.io/anjuna-unseal

        
<div class="section" id="introduction">
<h1>Introduction<a class="headerlink" href="#introduction" title="Permalink to this headline">¶</a></h1>
<div class="section" id="overview-of-hashicorp-vault">
<h2>Overview of Hashicorp Vault<a class="headerlink" href="#overview-of-hashicorp-vault" title="Permalink to this headline">¶</a></h2>
<p><a class="reference external" href="https://www.vaultproject.io/">Hashicorp Vault</a> is a popular tool for secrets
management, encryption as a service and privileged access management.</p>
<p>Organizations use Vault to ensure that secrets are not disseminated in
multiple places (configuration files, source control
management systems, scripts) and are only accessed by authorized parties <a class="footnote-reference" href="#id2" id="id1">[1]</a>.</p>
<p>To secure the secrets, Vault encrypts the data using a master key,
which is not stored on disk, but can be recreated in memory trough
a process called <em>Unsealing</em>. Instead of storing the master key, or distributing
it to administrators, Vault splits the master key into multiple key-shares
that can be used to recreate the  master key. This process, documented in
the <a class="reference external" href="https://www.vaultproject.io/docs/concepts/seal.html">Vault Seal/Unseal documentation</a>,
ensures that the master key is not stored at a single place (filesystem, database, etc.),
and makes it more difficult to compromise the master key.</p>
<p>However, since the master key is not readily available upon Vault startup
(or upon host reboot), automating the deployment of Vault securely becomes difficult
since coordinated manual operations are required on every node of a Vault cluster.</p>
<p>To enable such automation, Vault has added features to enable integration with
various HSM (Hardware Security Modules), or Cloud KMS (Key Management Systems),
which can be used to store the Vault master key.
Unfortunately, there are many deployment scenarios where HSM or KMS solutions are not available,
not allowed (due to reluctance to store the master key with the cloud provider), or too costly.</p>
<p>Moreover, such solutions often end up not secure, to the extent that the external storage of
the master key adds little value. The problem is that to fetch the master key
from the HSM (or Cloud KMS) Vault needs to authenticate itself to the HSM, and uses
credentials that are completely exposed to anyone with access to the host running Vault.
This is often referred to as the <strong>Secret-zero</strong> problem.</p>
<p>One example is using a PIN code to authenticate to an HSM: the PIN code is stored
in the clear in a configuration file (<strong>vault.hcl</strong>), and anyone with access to
this configuration file would be able to fetch the master key from the HSM pretending
to be the Vault server.</p>
<p>Anjuna Vault-Unseal tool provides a convenient and secure solution for “unsealing” Vault
by leveraging a novel <em>Secure Enclave</em> technology within
<a class="reference internal" href="https://docs.anjuna.io/vault-unseal/sgx-servers.html#intel-sgx-access"><span class="std std-ref">commodity Intel processors</span></a>.</p>
</div>
<div class="section" id="secure-enclaves">
<h2>Secure Enclaves<a class="headerlink" href="#secure-enclaves" title="Permalink to this headline">¶</a></h2>
<p>A best practice for securing sensitive applications and data is encryption.
While encrypting data at-rest and in-transit are well-recognized practices with
existing solutions, <em>runtime application security</em> has been an unsolved problem.
Plaintext code and data are exposed in memory at runtime, and that information
can include sensitive data and keys for data encryption at-rest and in-transit.
Data that is encrypted at-rest or in-transit needs to be decrypted in order to operate on it,
and at that point becomes vulnerable in the absence of runtime protection.</p>
<p>Anjuna provides runtime protection for sensitive applications and their data
regardless of the state of the infrastructure security, and without modifying
the source code or recompiling the application, thus seamlessly integrating
into existing DevOps processes.</p>
<p>It uses the following constructs provided by secure enclaves in modern processors:</p>
<ul class="simple">
<li><strong>Memory Isolation:</strong> Completely isolates the memory of an application from anything else on
that machine including the operating system.
Data never leaves the secure enclave unencrypted. No one can access the memory,
not even with root or physical access to the system.</li>
<li><strong>Remote Attestation:</strong> Ensures a secure trusted channel to back-end server application,
provides integrity by validating that expected code is running in the expected environment.</li>
</ul>
<p>Conventional approaches to securing applications have relied primarily on software
to provide protection.
However good the software implementation may be, an attacker that is able to
gain privileged access would be able to circumvent software defenses.
A recent and disruptive technology introduced in many new processor models
provides a better security and privacy model. It essentially enables to run an
application in an environment that is isolated from the host, while running on the same machine.
In the case of Intel®, the key enabler is the hardware-level memory isolation
introduced by Intel® Software Guard Extensions (SGX), that creates an encrypted partition of the memory.</p>
<p>Intel® Software Guard Extensions (Intel® SGX) brings a hardware root-of-trust
and program isolation to commodity processors, enabling handling of sensitive data in
a trusted execution environment called <em>secure enclave</em>.</p>
<div class="figure align-center" id="id3">
<img alt="Security Model" src="https://docs.anjuna.io/vault-unseal/_images/secmodel.png" style="width: 100%;" />
<p class="caption"><span class="caption-text">Secure enclaves enable running an application so that its contents and the data
it handles are completely inaccessible to any other entities.
Such “blocked” entities include privileged users on the guest OS,
the hypervisor or the host OS itself.</span></p>
</div>
</div>
<div class="section" id="anjuna-vault-unseal">
<h2>Anjuna Vault Unseal<a class="headerlink" href="#anjuna-vault-unseal" title="Permalink to this headline">¶</a></h2>
<p>Running the Vault Unsealing process in an Intel® SGX enclave provides the following
guarantees:</p>
<ul class="simple">
<li>The <em>Unseal Tokens</em> needed to unseal Vault are securely stored in an encrypted
configuration file by leveraging the Intel® SGX Data Sealing capabilities.</li>
<li>Unseal tokens are decrypted inside an SGX Enclave, which guarantees protection
against any memory scraping attempts.</li>
<li>The <cite>Anjuna Vault-Unseal tool</cite> uses TLS to communicate with Vault, ensuring authenticity,
confidentiality and integrity of all messages exchanged with Vault.
TLS termination inside the secure enclave ensures end-to-end security for the unseal tokens.</li>
</ul>
<p>After an initial setup to create and encrypt the Vault Unseal configuration
file, automating the unsealing process securely is easy:</p>
<ul class="simple">
<li>The unseal tokens can be stored encrypted on the file system to perform the
Vault unseal operation.</li>
<li>Only Anjuna Vault-Unseal tool running in an enclave can decrypt the unseal tokens.</li>
<li>The unsealing operation itself is completely secure, since it is running in
a secure enclave.</li>
<li>Unsealing is performed only when the Vault server is successfully authenticated.
Vault server certificate is pinned, and cannot be tampered with due to an integrity
protection provided by the secure enclave.</li>
<li>The encrypted Vault Unseal configuration file can be left on the host
without fear that it could be compromised.</li>
</ul>
<table class="docutils footnote" frame="void" id="id2" rules="none">
<colgroup><col class="label" /><col /></colgroup>
<tbody valign="top">
<tr><td class="label"><a class="fn-backref" href="#id1">[1]</a></td><td>See <a class="reference external" href="https://www.vaultproject.io/docs/">Hashicorp Vault Documentation</a> for more information on Vault.</td></tr>
</tbody>
</table>
</div>
</div>
