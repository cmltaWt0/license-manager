4. How Subscription Plan renewals are processed
===============================================

Status
======

* Proposed May 2021
* Accepted June 2021
* Updated to match reality August 2021
(This is all implemented and in production circa August 2021)

Context
=======

Defintions
----------

I'll try to be consistent about words used to refer to different plans below.  Generally:

* **renewal** The record, or processing thereof, referring to the fact that an existing
  subscription plan will be renewed into a new plan in the future.
* **original plan** The original plan, which will become expired at the end of the expiration process.
* **future plan** The new plan, created during the renewal process, and becomes effective some time after
  the renewal process is completed.

Background
----------
The previous ADR introduced the ``SubscriptionPlanRenewal`` model, which we've accepted/implemented.
It also proposed the idea of transferring licenses between the original plan and the future plan
as part of renewal processing, as in, we wouldn't create new ``License`` instances for the future plan,
but would instead update the existing licenses in the original plan to be associated with the future plan.

Upon further reflection, this was not the best plan and we didn't implement it.  The drawbacks to the plan are:

* It required duplicating data into a different system (``LicensedCourseEnrollment`` in ``edx-enterprise``)
  to capture the subscription plan identifier associated with the license at the time of enrollment.
* It decreased data reliability - we'd have to rely on historical records to understand a license's life-journey,
  rather than being able to reason about license history from the primary records,
  i.e. ``License`` and ``SubscriptionPlan`` records.  This would have limited our ability to expose historical
  license data to Enterprise Admins in the future.
* It would require funky business logic around **when** a renewal should be processed and require
  accepting that there would be "maintenance windows", even if only a few minutes long, during
  which a learner would not have an activated license in a current subscription plan.
* It increased the likihood of introducing data integrity issues during the renewal processing.
* It's more complex than the solution proposed below.

Instead, we'll create new
``License`` instances for the future plan when a renewal is processed.

Decision
========

The processing of a renewal is described by each of the subsections below.  We'll attempt
to design this process such that it's idempotent - that is, so that a ``SubscriptionPlanRenewal``
could be processed twice, and no undue changes would take effect or errors be raised on
the second processing.

Create a future SubscriptionPlan during renewal processing
----------------------------------------------------------
We'll create the future ``SubscriptionPlan`` record during the renewal process.  The renewal process
generally assumes that the future subscription plan does not already exist, but we'll make
some minimal effort toward designing the renewal process such that it is idempotent.
The renewal process will copy some fields from the original plan into the future plan, such as:

* ``customer_agreement``
* ``enterprise_catalog_uuid``
* ``is_active``
* ``netsuite_product_id``

What will the title of the future plan be?

* The future plan's title should be configurable.  We'll add a field to ``SubscriptionPlanRenewal``
  that allows a user in Django Admin to specify the title of the future plan.  The field will
  have a default value of ``{{Original Plan Title}} - Renewal ({{Year of future plan activation}})``

Create new License instances for the future plan
------------------------------------------------

We'll create as many licenses to be associated with the future plan as are requested in the
``SubscriptionPlanRenewal.number_of_licenses`` field.  We'll copy data from the original licenses
to the future licenses as appropriate.

Which licenses in the original plan should be "copied" to new licenses in the future plan?

* We won't copy revoked licenses.
* We'll add a field to ``SubscriptionPlanRenewal`` that allows a Django Admin user to specify
  what types of licenses to copy: ``activated and assigned``, ``activated only``, or ``none``, in
  that order.  The default value is ``activated and assigned``, meaning that we'll copy
  both ``activated`` and ``assigned`` license data to the future plan.
* The original license assignment date should be copied to the future license's assignment date.
  The ``created`` field on the record allows us to understand the distinction between when
  the database record was created and the assignment date from the end user's perspective.
* We'll copy the status value from the original license to the future license.  The future licenses
  will end up only being ``ASSIGNED`` or ``ACTIVATED``.
* We'll set a new ``License.renewed_to`` field to reference the ``uuid`` field of the future license.
  This field allows us to know the pre-renewal status of a license after the renewal
  is processed (e.g. was it "assigned" or "activated" before the renewal happened?).
  It will also let us directly reference an original license from a future license, or the converse.

The original licenses retain their status
-----------------------------------------

With the changes documented above, licensed learners may be in a state where they have
multiple licenses assigned to them - perhaps one in a current active plan, and another
in a "future" subscription plan that is active but not current - that is,
the future plan's ``start_date`` is in the future.

We'll modify the ``LearnerLicensesViewSet.base_queryset()`` method to lookup only
licenses for the requesting user who's plans are both active (``is_active=True``)
and current (the current date is within the plan's (start, expiration) date range) by default.
We'll introduce ``current_plans_only`` and ``active_plans_only`` optional query params
which allow the requester to disable that default behavior, so that licenses with
inactive or non-current plans are included in the response payload.

This modification also has the benefit of preventing the inclusion of certain licenses,
whose subscription plan is still marked as active but has an expiration date is in the past,
in the response payload this endpoint.

* We originally intended to introduce a new license status ``transferred-renewal`` to mark licenses as transferred 
  after a renewal was processed and the original plan was expired. We decided to reject this approach to 
  retain the original license statuses.

Expire subscription plans and licensed enrollments 
--------------------------------------------------
A cron job runs daily to execute the ``expire_subscriptions`` command which calls the Enterprise API Client to terminate course enrollments associated
with an expired license. Licensed enterprise course enrollments are changed to the "audit" course mode, or the user is unerolled from the 
course if no audit mode exists. The ``expiration_processed`` field on the subscription plan is set to True after the expiration has been processed.

No backfilling of data
----------------------

We won't have to tell edx-enterprise about any of this.  From its perspective, it doesn't
care if the same ``auth.User`` gets an enterprise enrollment associated with a new license UUID
and the same enterprise customer.

For downstream reporting, it's easy to trace from an ``EnterpriseEnrollment`` back to the subscription plan
which that enrollment should be attributed to, e.g.::

  EnterpriseCourseEnrollment -->
  LicensedCourseEnrollment -->
  License -->
  SubscriptionPlan -->
  CustomerAgreement

The ``license_uuid`` helps glue all of this together, because it can connect the ``LicensedCourseEnrollment``
with the ``SubscriptionPlan``.

Why this all works
------------------

From a learner's perspective, this all works because, in the learner portal, we
request enterprise enrollment records associated with that learner and enterprise, not on the basis
of license identifier.  As long as this remains true, which it should, this strategy
remains valid.  It should remain true because the existence of licenses is
hidden from the learner; licenses are only a background means of access control - there
are not UX elements that allow the user to read or update data about them.
