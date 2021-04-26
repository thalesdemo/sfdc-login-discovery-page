.. Salesforce Login Discovery Page documentation master file, created by
   sphinx-quickstart on Sat Apr 24 19:37:40 2021.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.


************************************
Salesforce Login Discovery Page
************************************

.. toctree::
   :hidden:
   :maxdepth: 2

   index


Overview
========

This guide explains how to vary the authentication behaviour to Salesforce
depending on the domain found in the username, by taking advantage of
Apex handlers.
By default, Salesforce does not have a built-in method to adapt the
authentication requirement for different users. This complicates integration
such as with Chatter, where external users may require a different form of
authentication than the other Salesforce users.

In this guide, we configure Salesforce in a way that only specific users will
use strong multi-factor authentication with SafeNet Trusted Access ("*STA*"), a
leading-class cybersecurity platform to control in real-time access patterns.

  - **Internal users** will login to Salesforce using *STA* as a 3\ :sup:`rd`
    party SAML SSO IdP, based on a specific list of domains ``domainFilters``
    that are compared against the ``Salesforce username``.
  - For **all other users**, authentication is based on the basic ``Salesforce
    password``.


Instructions
============

Salesforce Configuration
------------------------

.. note::
        The steps provided are using the Salesforce Lightning experience.
        While the steps may slightly vary in the Classic experience, the overall
        procedure remains the same.

.. _detailedsteps:

  #. From your **Salesforce admin console**, search or navigate to :guilabel:`Apex Classes`
     and click :kbd:`New`.

     .. thumbnail:: images/NavigateToCustomCodeApexClass.png
        :title: Figure: Navigate to the Apex Class menu in Salesforce

     |

  #. | Adjust and paste the :ref:`Apex handler <apex_handler>` code below, then click :kbd:`Save`.
     |

  #. Enable this custom login discovery handler under :menuselection:`Company Settings --> My Domain`.

      #. Under **Authentication Configuration** select the *Discovery Login Page* Type.
      #. For **Login Discovery Handler** select the :code:`DiscLoginSafeNetHandler` handler from the list of Apex classes.
      #. Also confirm that ``Login Form`` is checked as the *Authentication Service*.

         .. thumbnail:: images/ModifyMyDomainAuthConfig.png
            :title: Figure: Review your Authentication Configure settings
            :width: 50%

      #. Click on :kbd:`Save`.
      #. Verify your settings.

         .. thumbnail:: images/VerifySettingsDiscoveryDomain.png
            :title: Figure: Review your Authentication Configure settings


  #. **(Optional)** Enable the *Prevent login from \https://login.salesforce.com* policy under
     :menuselection:`Company Settings --> My Domain --> Policies`.
     Complete this step after validating the solution if this policy is not already enabled.



Apex Handler
""""""""""""

.. note::

   The following handler which implements a custom Login Discovery page is in
   Salesforce preview.

Parameters
""""""""""

The following parameters will need to be modified in the apex code to suit your environment:

``domainFilters``
      comma-separated list of domain names to be parsed from the Salesforce username
      for redirection to the *STA* IDP

``idpName``
      *API name* of the *STA* IDP in Salesforce



.. tip::
   The *API name* can be found under :menuselection:`Identity --> Single Sign-On Settings`
   by viewing the IDP config details ( ➝ click on the *STA* IDP name).

   .. thumbnail:: images/TipWhereToFindAPIName.png
      :title: Figure: Where to find the API name in Salesforce
      :width: 80%
      :align: center


.. _apex_handler:

Code [#]_
"""""""""

Refer to the definitions in the previous section to set the yellow highlighted parameters
for your own environment.

.. code-block:: obj-j
   :linenos:
   :emphasize-lines: 25, 40


   // This Salesforce Login Discovery class enables adaptive authentication logic from the domain
   // found in the Salesforce username (that is after an '@' symbol).
   // e.g. domain.com if incoming username is xyz@domain.com
   // Use Auth.DiscoveryCustomErrorException to throw custom errors which will be shown on login page.

   global class DiscLoginSafeNetHandler implements Auth.MyDomainLoginDiscoveryHandler {

      global PageReference login(String identifier, String startUrl, Map<String, String> requestAttributes)
      {
        if (identifier != null) {
          // Search for user by username
          List<User> users = [SELECT Id, Username FROM User WHERE Username = :identifier AND IsActive = TRUE];
          if (!users.isEmpty() && users.size() == 1) {
            return discoveryResult(users[0], startUrl, requestAttributes);
          } else {
            throw new Auth.LoginDiscoveryException('No unique user found. User count=' + users.size());
          }
        }
        throw new Auth.LoginDiscoveryException('Invalid Identifier');
      }

      private PageReference getSsoRedirect(User user, String startUrl, Map<String, String> requestAttributes)
      {
        // API name of the SAML IDP
        String idpName = 'idp';

        // Look up if the user should log in with IDP and return the URL to initialize SSO.
        SamlSsoConfig SSO = [select Id from SamlSsoConfig where DeveloperName=:idpName limit 1];

        // To get the URL for a My Domain subdomain, you can pass null in the communityURL parameter.
        String ssoUrl = Auth.AuthConfiguration.getSamlSsoUrl(null, startUrl, SSO.Id);
        return new PageReference(ssoUrl);
      }

      private PageReference discoveryResult(User user, String startUrl, Map<String, String> requestAttributes)
      {
        String domain = user.Username.split('@').get(1);

        // Modify the list of domains between the brackets. For single domain, do not include a comma separator.
        List<String> domainFilters = new List<String>{'thalesdemo.ml', 'thalesgroup.com'};

        PageReference ssoRedirect = null;
        try { ssoRedirect = getSsoRedirect(user, startUrl, requestAttributes); }
        catch(Exception e) { ssoRedirect = null; }

        if(ssoRedirect != null && domainFilters.contains(domain)) {
          return ssoRedirect;
        } else {
          return Auth.SessionManagement.finishLoginDiscovery(Auth.LoginDiscoveryMethod.password, user.Id);
        }
      }
   }




Validating the Solution
=======================

With the above ``domainFilters``, when the domain in the username is either *@thalesdemo.ml*
or *@thalesgroup.com*, users will be redirected to the STA IDP for authentication
to subsequently access Salesforce.

Other users will login with the regular Salesforce password. This is achieved by
changing the login experience on Salesforce. That is, instead of entering both
username and password on the first login window, the form now only asks for the username.

  #. | **Login** with ``admin@thalesdemo.ml``
     | ↳ **Result**: redirected to STA based on domain filter (long video)

     .. image:: images/TestLoginWithAdmin.gif
        :width: 85%
        :align: center
        :class: no-scaled-link

  #. | **Login** with ``sales@thalesgroup.com``
     | ↳ **Result**: redirected to STA based on domain filter (short video)

     .. image:: images/TestLoginWithSales.gif
        :width: 85%
        :align: center
        :class: no-scaled-link

  #. | **Login** with ``test-user@mailinator.com``
     | ↳ **Result**: prompted for Salesforce password

     .. image:: images/TestLoginWithExternal.gif
        :width: 85%
        :align: center
        :class: no-scaled-link

Troubleshooting
===============

You can debug apex classes with the ``System.Debug()`` statement. In order to
capture those logs, you first need to:

   #. Return to **Authentication Configuration** in :menuselection:`Company Settings --> My Domain` and click :kbd:`Edit`.

      |

   #. For **Execute Login As** select an administrator account and :kbd:`Save` the settings.

      .. thumbnail:: images/DebugAuthConfigExecuteLoginAs.png
         :title: Figure: Where to set the ExecuteLoginAs admin account that has manage user permissions
         :width: 60%

      |

   #. Navigate to  :menuselection:`Platform Tools --> Environments --> Logs` ‣ :guilabel:`Debug Logs` and click :kbd:`Edit`.

      .. thumbnail:: images/DebugAddNew.png
         :title: Figure: Where to add a trace flag
         :width: 80%

      |

   #. Configure the trace:

      +----------------------+------------------------------------------------------------------+
      | Property             | Value                                                            |
      +======================+==================================================================+
      | Trace Entity Type    | Select **User**                                                  |
      +----------------------+------------------------------------------------------------------+
      | Traced Entity Name   | Name of the admin account from the previous step                 |
      +----------------------+------------------------------------------------------------------+
      | Start Date           | Start time for the trace logs                                    |
      +----------------------+------------------------------------------------------------------+
      | Expiration Date      | Stop time for the trace logs                                     |
      +----------------------+------------------------------------------------------------------+
      | Debug Level          | Set or create a debug level (or higher) for *Apex Code* category |
      +----------------------+------------------------------------------------------------------+

      |

      .. thumbnail:: images/DebugAddTraceFlag.png
         :title: Figure: How to configure the trace
         :width: 80%

      |

   #. Modify the Apex Class Handler to add wherever desired ``System.Debug()`` statements.

      |

   #. Login to Salesforce (using the Discovery page) to generate logs.

      |

   #. Go back to  :menuselection:`Platform Tools --> Environments --> Logs` ‣ :guilabel:`Debug Logs` to view or download the logs.

      .. thumbnail:: images/DebugViewLogs.png
         :title: Figure: How to view or download the logs



-----------

.. rubric:: Footnotes

.. [#] | Salesforce code example
       | https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_interface_Auth_MyDomainLoginDiscoveryHandler.htm


.. rubric:: Contact

If you have any remarks about this guide, please don't hesitate to `contact us <mailto:alexander.basin@thalesgroup.com;cina.shaykhian@thalesgroup.com?subject=[Feedback]%20Salesforce%20Login%20Discovery%20Page>`_ directly !
