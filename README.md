# xml-signer

Provides signing and verification of XML documents for PHP focussing on support for signatures that are compliant with the [XAdES specification](https://www.etsi.org/deliver/etsi_en/319100_319199/31913201/01.01.01_60/en_31913201v010101p.pdf).

## XML-DSig

This project builds on [xmlseclibs](https://github.com/robrichards/xmlseclibs) by Rob Richards which provides [XML-DSig](https://www.w3.org/TR/xmldsig-core/) support.  However, this base project has been modified to accommodate requirements of XAdES.  For example but not limited to:

- the &lt;Reference> node referencing the XAdES &lt;QualifingProperties> needs to support a Type attribute
- more flexibility in the specification of a &lt;Reference> Id and URI attribute
- support added for [XPath Filter 2.0](https://www.w3.org/TR/xmldsig-filter2/)
- generate XML from a node list such as generated by an XPath query
- allow for building a certificate chain from certificates within a signature or from issuer links within a certificate 

Because these changes are important, the relevant source from [xmlseclibs](https://github.com/robrichards/xmlseclibs) has been incorporated into this project and the namespaces changed to prevent conflict with the original.

## XAdES support

There are two aspects to the XAdES support.  One is a set of classes that cover all the element types defined in the [XAdES specificiations](https://www.etsi.org/deliver/etsi_en/319100_319199/31913201/01.01.01_60/en_31913201v010101p.pdf).  This code is analogous to the [XAdESJS](https://github.com/PeculiarVentures/xadesjs) classes and is used in a similar way.  The class constructors accept appropriate parameters and are able to validate them. They are able to work together to create a hierarchy of nodes and the node hierarchy can be used to both load existing and generate new XML required for a valid signature.  There are also features to support navigating and manipulating the node hierarchy. Using these classes a person who knows XAdES is able to create arbitrary XAdES signatures.

However it is likely users will not have an encyclopaedic knowledge of XAdES.  So the other aspect is to allow less expert users provide a minimum amount of information and use functions to generate XML signatures, request and process time stamps and add counter signatures.  At the moment the handy functions do not cover the whole XAdES specification although they do allow use of the key XAdES features such as policy selection, commitment type indication, user roles, timestamps and counter signatures.  Signatures can be embedded in a source document or they can be saved as detached files.  A function is also available to validate existing signatures.

## Limitations

There are 140+ different elements defined by XAdES.  Although support exists for every one, the level of testing for each is not the same. This is particularly true of those in the &lt;UnsignedProperties> area.  Many of these elements are there to support non-repudiation over a long period of time.  They exist to cover cases such as validating a signature long after the certificate used to sign a document has expired.  I don't have an immediate requirement to use these features, nor do I have access to examples of the use of some elements so my ability to test them is limited. If you have a requirement to use these features of the specification and, in particular, to use what the specification refers to as archiving, and have a example cases, please get in touch as it will be good to expand testing.

Another limitation is that the only mechanism available to sign a signature is by using X509 certificates in the context of the public key infrastructure (PKI). These are the same type of certificates used by web sites and browers though usually with different key use attributes.

## Conformance

To check the generated XML conforms to the specification, they are verified using the [XAdES Conformance Checker (XAdESCC)](https://signatures-conformance-checker.etsi.org/pub/index.php) created by [Juan Carlos Cruellas](https://ieeexplore.ieee.org/author/37296299300). This tool has been created using Java and is provided via [ETSI](https://www.etsi.org/) which is the ESO responsible for the XAdES specification.  XAdESCC checks all aspects of a signature including the computed digests.  Using this tool helps to ensure a signature produced by one tool can be verified by another tool and vice-versa.  A C# program is also used to confirm generated signatures can be verified by the XML-DSig support included within the *System.Security.Cryptography.Xml* namespace.

## Dependencies

XAdES uses [timestamps]((https://datatracker.ietf.org/doc/html/rfc3161)), 
[Online Certificate Status Protocol (OCSP)](https://datatracker.ietf.org/doc/html/rfc6960) and 
[certificate revocation lists (CRLs)](https://datatracker.ietf.org/doc/html/rfc5280) to support signature non-repudiation.  
So in addition to XMLDSig, this [OCSP requester](https://github.com/bseddon/requester) project is used.  
It provides OCSP, timestamp protocol (TSP) and CRL request support. It also provides classes to read strings 
encoded using abstract syntax notation (ASN.1).  ASN.1 is used to encode OCSP and TSP requests and responses.  
It is also used to encode PKI entities such as X509 certificates, keys, stores and so on so ASN.1 is an important standard to support.

The [OCSP requester](https://github.com/bseddon/requester) relies on the following PHP extensions: php_curl, php_gmp, php_mbstring and php_openssl.

## Simple examples

Here's how to verify a document with a signature. The document here is on the end of a Url but it could be, is likely to be, a local file.

```php
XAdES::verifyDocument( 'http://www.xbrlquery.com/xades/hashes-signed.xml' );
```

Here's an example of signing an Xml document with a robust XAdES signature.  The url is to a real document but to be able to execute this example, it will be necessary to provide your own certificate and corresponding private key.  The output of this function when used with my test certificate and and private key [can be accessed here](http://www.xbrlquery.com/xades/hashes%20for%20nba%20with%20signature.xml).

```php

use lyquidity\xmldsig\CertificateResourceInfo;
use lyquidity\xmldsig\InputResourceInfo;
use lyquidity\xmldsig\KeyResourceInfo;
use lyquidity\xmldsig\ResourceInfo;
use lyquidity\xmldsig\XAdES;
use lyquidity\xmldsig\xml\SignatureProductionPlaceV2;
use lyquidity\xmldsig\xml\SignerRoleV2;
use lyquidity\xmldsig\XMLSecurityDSig;

XAdES::signDocument( 
	new InputResourceInfo(
		'http://www.xbrlquery.com/xades/hashes for nba.xml', // The source document
		ResourceInfo::url, // The source is a url
		__DIR__, // The location to save the signed document
		'hashes for nba with signature.xml' // The name of the file to save the signed document in
	),
	new CertificateResourceInfo( '...some path to a signing certificate...', ResourceInfo::file ),
	new KeyResourceInfo( '...some path to a correspondoing private key...', ResourceInfo::file ),
	new SignatureProductionPlaceV2(
		'My city',
		'My address', // This is V2 only
		'My region',
		'My postcode',
		'My country code'
	),
	new SignerRoleV2(
		'CEO'
	),
	array(
		'canonicalizationMethod' => XMLSecurityDSig::C14N,
		'addTimestamp' => false // Include a timestamp? Can specify an alternative TSA url eg 'http://mytsa.com/' 
	)
);
```

## Minimal input

A goal of the design of the classes is to try to minimize input.  There are several cases in this example.  The parameters to the **SignatureProductionPlaceV2** class are really classes, for example, City, StreetAddress, Postcode, etc..  Constructors will convert a string paramter to an appropriate class instance if possible.

Another case in the example is **new SignerRoleV2('CEO')**. The full format of this is would be:

```php
new SignerRoleV2(
	new ClaimedEoles(
		array(
			new ClaimedRole('CEO')
		)
	)
)
```

Obviously the longer form will need to be used if and when there is more than one claimed role.

If the file references are just paths to a file then just the file path can be used.  In this example, this will apply to the certificate and private key reference.  This simplification cannot be used for the reference to the document to be signed because the source is a URL. As the generated signature cannot be saved into the same location in this case, an alternative location has to be supplied explicitly by instantiating the **InputResourceInfo** class and assigning values to the **saveLocation** and **saveFilename** properties.  This will also need to be done if the source is a local file that is read-only.

## Policies

In the examples above, the XAdES class is used directly.  However, this is unlikely to be too helpful as XAdES signatures are designed to be used with policies. That is, verifying a XAdES signature is not just a case of testing the XML-DSig <SignatureValue>.

The purpose of XAdES is to extend the core XML-DSig specification so that signed documents are able to withstand scrutiny in a court case.  In this way, XML signatures can have the same legal status as a written signature.  However, what constitutes adequate evidence of a signature for one jurisdiction may not be adequate for another.  To help make all parties operating within a juridiction aware of the requirements, each jurisdiction is able to publish a policy (another signed XML document) to describe their requirements.

There's no single definition of a jurisdiction.  An obvious jurisdiction might be a country or a govenment department.  However, it might also be a collection of commerical entities that reach a mutual agreement regarding the types of signatures that will be accepted.

An example policy is the one created by the [Netherlands Standard Business Reporting (SBR)](https://business.gov.nl/regulation/standard-business-reporting/) which is the department of the Netherlands government that receives and checks returns produced by commerical entities operating in the Netherlands.  This department accepts electronic returns in the form of [XBRL](https://www.xbrl.org) documents.  These documents can be signed by auditors. The information to be included in the signature, the key length to be used by certificates and so on is defined in [this XML policy document](http://nltaxonomie.nl/sbr/signature_policy_schema/v2.0/SBR-signature-policy-v2.0.xml).

It is important then, when validating a signature the validation checks not only the digest created by applying the certificate, but also that the components included in the signature XML conform with the relevant policy.  It is better, then, that a jurisdiction specific class is created, one that extends the class **XAdES**.

Included with the project is such a class for the [SBR jurisdiction policy](http://nltaxonomie.nl/sbr/signature_policy_schema/v2.0/SBR-signature-policy-v2.0.xml). It is called **XAdES_SBR**.  It overrides functions on **XAdES** both to add extra, policy relevant, information to the signature, and test that appropriate information is included in the signature when it is validated.

## Resources

In the example above three types of resource are used.  Each is an instance type specific to the type of resource being required for a parameter.  The resources are used to convey appropriate information to the signer and this information can be in different forms.  For example, the resource for the document to be signed might be a file path, a url, a string of XML or a DOMDocument instance.  Equally the resource for a signature or private key might be file path, a PEM formatted string, a base 64 encoded string or a binary representation of the certificate or key.

As well as having resource specific properties, each has a 'type' property that provides a way to describe the nature of the resource.  The complete list of possible 

|Flag|A flag indicating the resource is a... |
|-|-|
|file|Path to a local file.  If no explicit type is provided this is the default type|
|string|String|
|binary|Binary string|
|der|DER encoded binary string|
|xmlDocument|A DOMDocument instance|
|pem|PEM encoded string|
|url|URL to the document source|
|base64|Base64 encoded|

Where appropriate these flags can be or'd together.  Doing this means more information can be provided to the signer.  For example a certificate provided in a string might be in a PEM format, a Base 64 format or as DER bytes.  If a certificate is provided in a string in a PEM format the type value will be:

```php
ResourceInfo::string | ResourceInfo::pem
```

## How to Install

Install with [`composer.phar`](http://getcomposer.org).

```sh
php composer.phar require "lyquidity/xml-signer"
```

# References

The two main XAdES specifications are linked below. 

[ETSI EN 319 132-1 V1.1.1 (2016-04)](https://www.etsi.org/deliver/etsi_en/319100_319199/31913201/01.01.01_60/en_31913201v010101p.pdf)

[ETSI EN 319 132-2 V1.1.1 (2016-04)](https://www.etsi.org/deliver/etsi_en/319100_319199/31913202/01.01.01_60/en_31913202v010101p.pdf)

These are not the only XAdES specifications but the others are no longer applicable.  The link below is to a question and response from [Juan Carlos Cruellas](https://ieeexplore.ieee.org/author/37296299300), someone involved in the development of the specfications.

[Specification History](https://github.com/lovele0107/signatures-conformance-checker/issues/21)

Fortunately, Juan Carlos Cruellas has created a really useful tool to perform conformance tests against signatures created by any tool.

[XAdES Conformance Checker](https://signatures-conformance-checker.etsi.org/pub/index.php)

XAdES is reliant on other specificiations.  It is directly dependent on XMLDSig.  This in turn is reliant other XML specifications.

[XMLDSig](https://www.w3.org/TR/xmldsig-core/)

[XPath Filter 2.0](https://www.w3.org/TR/xmldsig-filter2/)

[XML Canonicalization](https://www.w3.org/TR/xml-c14n/)

[XML Exclusive Canonicalization](https://www.w3.org/TR/xml-exc-c14n/)

XAdES requires timestamps to provide non-repudiation.  Requests, responses and their contents are defined by IETF RFCs.

[PKI Timestamp protocol [rfc3161]](https://datatracker.ietf.org/doc/html/rfc3161) updated by [[rfc5816]](https://www.ietf.org/rfc/rfc5816.txt)

Certificates used to sign signatures have to be checked for validity.  One mechanism to check the status of certificate is Online Certificate Status Protocol (OCSP)

[OCSP [rfc6960]](https://datatracker.ietf.org/doc/html/rfc6960)
