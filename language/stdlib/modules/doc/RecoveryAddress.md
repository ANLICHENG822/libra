
<a name="0x1_RecoveryAddress"></a>

# Module `0x1::RecoveryAddress`

### Table of Contents

-  [Resource `RecoveryAddress`](#0x1_RecoveryAddress_RecoveryAddress)
-  [Function `publish`](#0x1_RecoveryAddress_publish)
-  [Function `rotate_authentication_key`](#0x1_RecoveryAddress_rotate_authentication_key)
-  [Function `add_rotation_capability`](#0x1_RecoveryAddress_add_rotation_capability)
-  [Specification](#0x1_RecoveryAddress_Specification)
    -  [Module specifications](#0x1_RecoveryAddress_@Module_specifications)
        -  [RecoveryAddress has its own KeyRotationCapability](#0x1_RecoveryAddress_@RecoveryAddress_has_its_own_KeyRotationCapability)
        -  [RecoveryAddress resource stays](#0x1_RecoveryAddress_@RecoveryAddress_resource_stays)
        -  [RecoveryAddress remains same](#0x1_RecoveryAddress_@RecoveryAddress_remains_same)
        -  [Only VASPs can be RecoveryAddress](#0x1_RecoveryAddress_@Only_VASPs_can_be_RecoveryAddress)
    -  [Function `publish`](#0x1_RecoveryAddress_Specification_publish)
    -  [Specifications for individual functions](#0x1_RecoveryAddress_@Specifications_for_individual_functions)
    -  [Function `rotate_authentication_key`](#0x1_RecoveryAddress_Specification_rotate_authentication_key)
    -  [Function `add_rotation_capability`](#0x1_RecoveryAddress_Specification_add_rotation_capability)



<a name="0x1_RecoveryAddress_RecoveryAddress"></a>

## Resource `RecoveryAddress`

A resource that holds the
<code>KeyRotationCapability</code>s for several accounts belonging to the
same VASP. A VASP account that delegates its
<code>KeyRotationCapability</code> to
but also allows the account that stores the
<code><a href="#0x1_RecoveryAddress">RecoveryAddress</a></code> resource to rotate its
authentication key.
This is useful as an account recovery mechanism: VASP accounts can all delegate their
rotation capabilities to a single
<code><a href="#0x1_RecoveryAddress">RecoveryAddress</a></code> resource stored under address A.
The authentication key for A can be "buried in the mountain" and dug up only if the need to
recover one of accounts in
<code>rotation_caps</code> arises.


<pre><code><b>resource</b> <b>struct</b> <a href="#0x1_RecoveryAddress">RecoveryAddress</a>
</code></pre>



<details>
<summary>Fields</summary>


<dl>
<dt>

<code>rotation_caps: vector&lt;<a href="LibraAccount.md#0x1_LibraAccount_KeyRotationCapability">LibraAccount::KeyRotationCapability</a>&gt;</code>
</dt>
<dd>

</dd>
</dl>


</details>

<a name="0x1_RecoveryAddress_publish"></a>

## Function `publish`

Extract the
<code>KeyRotationCapability</code> for
<code>recovery_account</code> and publish it in a
<code><a href="#0x1_RecoveryAddress">RecoveryAddress</a></code> resource under
<code>recovery_account</code>.
Aborts if
<code>recovery_account</code> has delegated its
<code>KeyRotationCapability</code>, already has a
<code><a href="#0x1_RecoveryAddress">RecoveryAddress</a></code> resource, or is not a VASP.


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_publish">publish</a>(recovery_account: &signer, rotation_cap: <a href="LibraAccount.md#0x1_LibraAccount_KeyRotationCapability">LibraAccount::KeyRotationCapability</a>)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_publish">publish</a>(recovery_account: &signer, rotation_cap: KeyRotationCapability) {
    // Only VASPs can create a recovery address
    <b>assert</b>(<a href="VASP.md#0x1_VASP_is_vasp">VASP::is_vasp</a>(<a href="Signer.md#0x1_Signer_address_of">Signer::address_of</a>(recovery_account)), ENOT_A_VASP);
    // put the rotation capability for the recovery account itself in `rotation_caps`. This
    // <b>ensures</b> two things:
    // (1) It's not possible <b>to</b> get into a "recovery cycle" where A is the recovery account for
    //     B and B is the recovery account for A
    // (2) rotation_caps is always nonempty
    <b>assert</b>(*<a href="LibraAccount.md#0x1_LibraAccount_key_rotation_capability_address">LibraAccount::key_rotation_capability_address</a>(&rotation_cap)
         == <a href="Signer.md#0x1_Signer_address_of">Signer::address_of</a>(recovery_account), EKEY_ROTATION_DEPENDENCY_CYCLE);
    move_to(
        recovery_account,
        <a href="#0x1_RecoveryAddress">RecoveryAddress</a> { rotation_caps: <a href="Vector.md#0x1_Vector_singleton">Vector::singleton</a>(rotation_cap) }
    )
}
</code></pre>



</details>

<a name="0x1_RecoveryAddress_rotate_authentication_key"></a>

## Function `rotate_authentication_key`

Rotate the authentication key of
<code>to_recover</code> to
<code>new_key</code>. Can be invoked by either
<code>recovery_address</code> or
<code>to_recover</code>.
Aborts if
<code>recovery_address</code> does not have the
<code>KeyRotationCapability</code> for
<code>to_recover</code>.


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_rotate_authentication_key">rotate_authentication_key</a>(account: &signer, recovery_address: address, to_recover: address, new_key: vector&lt;u8&gt;)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_rotate_authentication_key">rotate_authentication_key</a>(
    account: &signer,
    recovery_address: address,
    to_recover: address,
    new_key: vector&lt;u8&gt;
) <b>acquires</b> <a href="#0x1_RecoveryAddress">RecoveryAddress</a> {
    // Check that `recovery_address` has a `<a href="#0x1_RecoveryAddress">RecoveryAddress</a>` <b>resource</b>
    <b>assert</b>(exists&lt;<a href="#0x1_RecoveryAddress">RecoveryAddress</a>&gt;(recovery_address), ENOT_A_RECOVERY_ADDRESS);
    <b>let</b> sender = <a href="Signer.md#0x1_Signer_address_of">Signer::address_of</a>(account);
    <b>assert</b>(
        // The original owner of a key rotation capability can rotate its own key
        sender == to_recover ||
        // The owner of the `<a href="#0x1_RecoveryAddress">RecoveryAddress</a>` <b>resource</b> can rotate any key
        sender == recovery_address,
        ECANNOT_ROTATE_KEY
    );

    <b>let</b> caps = &borrow_global&lt;<a href="#0x1_RecoveryAddress">RecoveryAddress</a>&gt;(recovery_address).rotation_caps;
    <b>let</b> i = 0;
    <b>let</b> len = <a href="Vector.md#0x1_Vector_length">Vector::length</a>(caps);
    <b>while</b> ({
        <b>spec</b> {
            <b>assert</b> i &lt;= len;
            <b>assert</b> forall j in 0..i: caps[j].account_address != to_recover;
        };
        (i &lt; len)
    })
    {
        <b>let</b> cap = <a href="Vector.md#0x1_Vector_borrow">Vector::borrow</a>(caps, i);
        <b>if</b> (<a href="LibraAccount.md#0x1_LibraAccount_key_rotation_capability_address">LibraAccount::key_rotation_capability_address</a>(cap) == &to_recover) {
            <a href="LibraAccount.md#0x1_LibraAccount_rotate_authentication_key">LibraAccount::rotate_authentication_key</a>(cap, new_key);
            <b>return</b>
        };
        i = i + 1
    };
    <b>spec</b> {
        <b>assert</b> i == len;
        <b>assert</b> forall j in 0..len: caps[j].account_address != to_recover;
    };
    // Couldn't find `to_recover` in the account recovery <b>resource</b>; <b>abort</b>
    <b>abort</b> EACCOUNT_NOT_RECOVERABLE
}
</code></pre>



</details>

<a name="0x1_RecoveryAddress_add_rotation_capability"></a>

## Function `add_rotation_capability`

Add
<code>to_recover</code> to the
<code><a href="#0x1_RecoveryAddress">RecoveryAddress</a></code> resource under
<code>recovery_address</code>.
Aborts if
<code>to_recover.address</code> and
<code>recovery_address belong <b>to</b> different VASPs, or <b>if</b>
</code>recovery_address
<code> does not have a </code>RecoveryAddress` resource.


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_add_rotation_capability">add_rotation_capability</a>(to_recover: <a href="LibraAccount.md#0x1_LibraAccount_KeyRotationCapability">LibraAccount::KeyRotationCapability</a>, recovery_address: address)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_add_rotation_capability">add_rotation_capability</a>(to_recover: KeyRotationCapability, recovery_address: address)
<b>acquires</b> <a href="#0x1_RecoveryAddress">RecoveryAddress</a> {
    // Check that `recovery_address` has a `<a href="#0x1_RecoveryAddress">RecoveryAddress</a>` <b>resource</b>
    <b>assert</b>(exists&lt;<a href="#0x1_RecoveryAddress">RecoveryAddress</a>&gt;(recovery_address), ENOT_A_RECOVERY_ADDRESS);
    // Only accept the rotation capability <b>if</b> both accounts belong <b>to</b> the same <a href="VASP.md#0x1_VASP">VASP</a>
    <b>let</b> to_recover_address = *<a href="LibraAccount.md#0x1_LibraAccount_key_rotation_capability_address">LibraAccount::key_rotation_capability_address</a>(&to_recover);
    <b>assert</b>(
        <a href="VASP.md#0x1_VASP_parent_address">VASP::parent_address</a>(recovery_address) == <a href="VASP.md#0x1_VASP_parent_address">VASP::parent_address</a>(to_recover_address),
        EINVALID_KEY_ROTATION_DELEGATION
    );

    <a href="Vector.md#0x1_Vector_push_back">Vector::push_back</a>(
        &<b>mut</b> borrow_global_mut&lt;<a href="#0x1_RecoveryAddress">RecoveryAddress</a>&gt;(recovery_address).rotation_caps,
        to_recover
    );
}
</code></pre>



</details>

<a name="0x1_RecoveryAddress_Specification"></a>

## Specification


<a name="0x1_RecoveryAddress_@Module_specifications"></a>

### Module specifications



<pre><code>pragma verify = <b>true</b>;
</code></pre>



Returns true if
<code>addr</code> is a recovery address.


<a name="0x1_RecoveryAddress_spec_is_recovery_address"></a>


<pre><code><b>define</b> <a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(addr: address): bool
{
    exists&lt;<a href="#0x1_RecoveryAddress">RecoveryAddress</a>&gt;(addr)
}
</code></pre>


Returns all the
<code>KeyRotationCapability</code>s held at
<code>recovery_address</code>.


<a name="0x1_RecoveryAddress_spec_get_rotation_caps"></a>


<pre><code><b>define</b> <a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address: address):
    vector&lt;<a href="LibraAccount.md#0x1_LibraAccount_KeyRotationCapability">LibraAccount::KeyRotationCapability</a>&gt;
{
    <b>global</b>&lt;<a href="#0x1_RecoveryAddress">RecoveryAddress</a>&gt;(recovery_address).rotation_caps
}
</code></pre>


Returns true if
<code>recovery_address</code> holds the
<code>KeyRotationCapability</code> for
<code>addr</code>.


<a name="0x1_RecoveryAddress_spec_holds_key_rotation_cap_for"></a>


<pre><code><b>define</b> <a href="#0x1_RecoveryAddress_spec_holds_key_rotation_cap_for">spec_holds_key_rotation_cap_for</a>(
    recovery_address: address,
    addr: address): bool
{
    // BUG: the commented out version will <b>break</b> the postconditions.
    // exists i in 0..len(<a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address)):
    //     <a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address)[i].account_address == addr
    exists i: u64
        where 0 &lt;= i && i &lt; len(<a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address)):
            <a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address)[i].account_address == addr
}
</code></pre>



<a name="0x1_RecoveryAddress_@RecoveryAddress_has_its_own_KeyRotationCapability"></a>

#### RecoveryAddress has its own KeyRotationCapability



<a name="0x1_RecoveryAddress_RecoveryAddressHasItsOwnKeyRotationCap"></a>


<pre><code><b>schema</b> <a href="#0x1_RecoveryAddress_RecoveryAddressHasItsOwnKeyRotationCap">RecoveryAddressHasItsOwnKeyRotationCap</a> {
    <b>invariant</b> <b>module</b> forall addr1: address
        where <a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(addr1):
            len(<a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(addr1)) &gt; 0
            && <a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(addr1)[0].account_address == addr1;
}
</code></pre>




<pre><code><b>apply</b> <a href="#0x1_RecoveryAddress_RecoveryAddressHasItsOwnKeyRotationCap">RecoveryAddressHasItsOwnKeyRotationCap</a> <b>to</b> *;
</code></pre>



<a name="0x1_RecoveryAddress_@RecoveryAddress_resource_stays"></a>

#### RecoveryAddress resource stays



<a name="0x1_RecoveryAddress_RecoveryAddressStays"></a>


<pre><code><b>schema</b> <a href="#0x1_RecoveryAddress_RecoveryAddressStays">RecoveryAddressStays</a> {
    <b>ensures</b> forall addr1: address:
        <b>old</b>(<a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(addr1))
        ==&gt; <a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(addr1);
}
</code></pre>




<pre><code><b>apply</b> <a href="#0x1_RecoveryAddress_RecoveryAddressStays">RecoveryAddressStays</a> <b>to</b> *;
</code></pre>



<a name="0x1_RecoveryAddress_@RecoveryAddress_remains_same"></a>

#### RecoveryAddress remains same



<a name="0x1_RecoveryAddress_RecoveryAddressRemainsSame"></a>


<pre><code><b>schema</b> <a href="#0x1_RecoveryAddress_RecoveryAddressRemainsSame">RecoveryAddressRemainsSame</a> {
    <b>ensures</b> forall recovery_addr: address, to_recovery_addr: address
        where <b>old</b>(<a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(recovery_addr)):
            <b>old</b>(<a href="#0x1_RecoveryAddress_spec_holds_key_rotation_cap_for">spec_holds_key_rotation_cap_for</a>(recovery_addr, to_recovery_addr))
            ==&gt; <a href="#0x1_RecoveryAddress_spec_holds_key_rotation_cap_for">spec_holds_key_rotation_cap_for</a>(recovery_addr, to_recovery_addr);
}
</code></pre>




<pre><code><b>apply</b> <a href="#0x1_RecoveryAddress_RecoveryAddressRemainsSame">RecoveryAddressRemainsSame</a> <b>to</b> *;
</code></pre>



<a name="0x1_RecoveryAddress_@Only_VASPs_can_be_RecoveryAddress"></a>

#### Only VASPs can be RecoveryAddress



<a name="0x1_RecoveryAddress_RecoveryAddressIsVASP"></a>


<pre><code><b>schema</b> <a href="#0x1_RecoveryAddress_RecoveryAddressIsVASP">RecoveryAddressIsVASP</a> {
    <b>invariant</b> <b>module</b> forall recovery_addr: address
        where <a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(recovery_addr):
            <a href="VASP.md#0x1_VASP_spec_is_vasp">VASP::spec_is_vasp</a>(recovery_addr);
}
</code></pre>




<pre><code><b>apply</b> <a href="#0x1_RecoveryAddress_RecoveryAddressIsVASP">RecoveryAddressIsVASP</a> <b>to</b> *;
</code></pre>



<a name="0x1_RecoveryAddress_Specification_publish"></a>

### Function `publish`


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_publish">publish</a>(recovery_account: &signer, rotation_cap: <a href="LibraAccount.md#0x1_LibraAccount_KeyRotationCapability">LibraAccount::KeyRotationCapability</a>)
</code></pre>



<a name="0x1_RecoveryAddress_@Specifications_for_individual_functions"></a>

### Specifications for individual functions



<pre><code><b>aborts_if</b> !<a href="VASP.md#0x1_VASP_spec_is_vasp">VASP::spec_is_vasp</a>(<a href="Signer.md#0x1_Signer_spec_address_of">Signer::spec_address_of</a>(recovery_account));
<b>aborts_if</b> <a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(<a href="Signer.md#0x1_Signer_spec_address_of">Signer::spec_address_of</a>(recovery_account));
<b>aborts_if</b> <a href="LibraAccount.md#0x1_LibraAccount_spec_key_rotation_capability_address">LibraAccount::spec_key_rotation_capability_address</a>(rotation_cap) != <a href="Signer.md#0x1_Signer_spec_address_of">Signer::spec_address_of</a>(recovery_account);
<b>ensures</b> <a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(<a href="Signer.md#0x1_Signer_spec_address_of">Signer::spec_address_of</a>(recovery_account));
</code></pre>



<a name="0x1_RecoveryAddress_Specification_rotate_authentication_key"></a>

### Function `rotate_authentication_key`


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_rotate_authentication_key">rotate_authentication_key</a>(account: &signer, recovery_address: address, to_recover: address, new_key: vector&lt;u8&gt;)
</code></pre>




<pre><code><b>aborts_if</b> !<a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(recovery_address);
<b>aborts_if</b> !exists&lt;<a href="LibraAccount.md#0x1_LibraAccount_LibraAccount">LibraAccount::LibraAccount</a>&gt;(to_recover);
<b>aborts_if</b> len(new_key) != 32;
<b>aborts_if</b> !<a href="#0x1_RecoveryAddress_spec_holds_key_rotation_cap_for">spec_holds_key_rotation_cap_for</a>(recovery_address, to_recover);
<b>aborts_if</b> !(<a href="Signer.md#0x1_Signer_spec_address_of">Signer::spec_address_of</a>(account) == recovery_address
            || <a href="Signer.md#0x1_Signer_spec_address_of">Signer::spec_address_of</a>(account) == to_recover);
<b>ensures</b> <b>global</b>&lt;<a href="LibraAccount.md#0x1_LibraAccount_LibraAccount">LibraAccount::LibraAccount</a>&gt;(to_recover).authentication_key == new_key;
</code></pre>



<a name="0x1_RecoveryAddress_Specification_add_rotation_capability"></a>

### Function `add_rotation_capability`


<pre><code><b>public</b> <b>fun</b> <a href="#0x1_RecoveryAddress_add_rotation_capability">add_rotation_capability</a>(to_recover: <a href="LibraAccount.md#0x1_LibraAccount_KeyRotationCapability">LibraAccount::KeyRotationCapability</a>, recovery_address: address)
</code></pre>




<pre><code><b>aborts_if</b> !<a href="#0x1_RecoveryAddress_spec_is_recovery_address">spec_is_recovery_address</a>(recovery_address);
<b>aborts_if</b> <a href="VASP.md#0x1_VASP_spec_parent_address">VASP::spec_parent_address</a>(recovery_address) != <a href="VASP.md#0x1_VASP_spec_parent_address">VASP::spec_parent_address</a>(<a href="LibraAccount.md#0x1_LibraAccount_spec_key_rotation_capability_address">LibraAccount::spec_key_rotation_capability_address</a>(to_recover));
<b>ensures</b> <a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address)[
    len(<a href="#0x1_RecoveryAddress_spec_get_rotation_caps">spec_get_rotation_caps</a>(recovery_address)) - 1] == to_recover;
</code></pre>
