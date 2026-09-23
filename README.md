# Ztest1
************************************************************************
* 4/2/26   smartShift project
* Rules applied:  601 RUN 1
************************************************************************
METHOD create_payment_specification.
************************************************************************
* Author      : Kaisar Kazi Net Id: CG17118
* Date        : 13.05.2021
* Reference   : New Build
* Transport/RT: ARDK918845 / 8719
* FS No       : D812
* Description : Repayment payment specification creation from CESA
************************************************************************
* Author      : Debajyoti Das Net Id: CG15074
* Date        : 01.07.2021
* Reference   : ITSA-R6-Unauthorized agent fix
* Transport/RT: ARDK919933 / 8719
* FS No       : D812
* Description : Repayment not triggered for unauthorized agent
************************************************************************
* Author      : Suchitra N L Net Id: CG23603
* Date        : 22.07.2026
* Reference   : ARXK961808
* Transport/RT: ARXK961808/ RT 14749
* FS No       : D1517
* Description : ZPAY to pick only SA return charges
************************************************************************
* Author      : Navya Narakatla Net Id: CG23604
* Date        : 21.09.2026
* Reference   : ARXK963586
* Transport/RT: ARXK963586/ RT 12134
* FS No       : D1517
* Description : Added PSOBTYP to SELECT so the existing exclusion
*               check works correctly
************************************************************************

  DATA: lv_pdkey       TYPE pdkey_kk,
        lv_tot_amount  TYPE betrw_kk VALUE IS INITIAL,
        ls_log         TYPE bapiret2,
        ls_dfkkip_grp  TYPE zsfi_acc_info_repayment_create,
        lr_proc        TYPE REF TO zcl_itsa_repayment_processing,
        ls_dfkkip_item TYPE zsfi_acc_info_repayments_items,
        lt_dfkkip_item TYPE STANDARD TABLE OF zsfi_acc_info_repayments_items.
*Start of changes for ARXK961808
  TYPES: BEGIN OF ty_trans,
           main_trans TYPE hvorg_kk,
           sub_trans  TYPE tvorg_kk,
         END OF ty_trans.

  DATA lt_trans TYPE TABLE OF ty_trans.
*End of changes for ARXK961808
  CONSTANTS: lc_pdtyp_2          TYPE pdtyp_kk VALUE '2',
             lc_tot_amount       TYPE betrw_kk VALUE 10,
             lc_functional_error TYPE zdeerror_type VALUE 'F',
             lc_itsa_msgid       TYPE sy-msgid VALUE 'ZETMP_ITSA',
             lc_no_utr           TYPE sy-msgno VALUE '025',
             lc_no_credit        TYPE sy-msgno VALUE '026',
             lc_pymethod         TYPE sy-msgno VALUE '027',
             lc_exceed_10        TYPE sy-msgno VALUE '029',
             lc_ps_ret_org       TYPE zde_rep_origin VALUE 'R',
             lc_regime_itsa      TYPE zregime VALUE 'ITSA'. " ARXK961808

  CLEAR : es_error_log,ev_pdkey,es_payment_spec_error.

*  IF is_form_bundle_header-taxpayer IS NOT INITIAL AND is_form_bundle_header-account IS NOT INITIAL.
** Validation 1:__________C_ACCOUNT_NUMBER is recieved from API
** iv_acc_num is not initial.
*
** Validation 2:__________UTR number is available for BP
*fetch utr for a bp
  SELECT              idnumber
                 FROM but0id
                 WHERE partner = @iv_gpart
                 AND type = @zcl_safe_constants=>gc_utr
                 ORDER BY PRIMARY KEY                                                            "$smart: #601
                 INTO @DATA(lv_utr) UP TO 1 ROWS.                                                "$smart: #601
  ENDSELECT.
  IF sy-subrc NE 0.

    es_payment_spec_error-error_type = lc_functional_error.
    es_payment_spec_error-error_desc = TEXT-001.

    es_error_log-id    = lc_itsa_msgid.
    es_error_log-type   = zcl_safe_constants=>gc_error.
    es_error_log-number = lc_no_utr.


    RETURN.
  ENDIF.

** Validation 3:__________Check for Credit Items for ITSA CO

*****(need logic for payment lock, clearing lock)

*Get respective DFKKOP_CHARGES details
  SELECT a~opbel, a~opupw, a~opupk, a~opupz, a~applk, a~hvorg, a~tvorg, a~faedn, a~betrw, a~abrzu, a~abrzo,
         a~pdtyp, a~ikey, a~zzbnky, a~zzbnkn, a~zzpble_ord, a~zzunq_trans_id, a~zzpay_source, a~zzcard_type,a~vktyp_ps, a~stakz,a~psobtyp, b~txt30 "ARXK961808
    FROM dfkkop       AS a
   INNER JOIN tfkhvot AS b                             "#EC CI_BUFFJOIN
         ON a~hvorg = b~hvorg
     INTO TABLE @DATA(lt_dfkkop)
    WHERE augst = @zcl_safe_constants=>gc_augst_openitem
      AND gpart = @iv_gpart
      AND vkont = @iv_vkont
      AND spzah = @space     " Lock Reason for Automatic Payment
      AND betrw < 0
      AND pdtyp NE @lc_pdtyp_2
      AND stakz = @space                  "ARXK961808
      AND b~spras = @sy-langu.

*Start of changes for ARXK961808

* Filter out non-relevant ITSA credit items before payment specification creation
  IF lt_dfkkop IS NOT INITIAL.
* Get the Contract Account Category configured for ITSA regime
    SELECT vktyp FROM ztregime2cacorev  WHERE regime = @lc_regime_itsa ORDER BY regime, vktyp INTO @DATA(lv_vktyp)
      UP TO 1 ROWS.
    ENDSELECT.
* Keep only credit items belonging to the relevant ITSA Contract Account Category
    IF sy-subrc EQ 0.
      DELETE lt_dfkkop WHERE vktyp_ps <> lv_vktyp.
    ENDIF.
  ENDIF.

* Exclude configured non-repayable main/sub transaction combinations from repayment processing
  IF lv_vktyp IS NOT INITIAL AND lt_dfkkop IS NOT INITIAL.
    SELECT a~main_trans, a~sub_trans
          FROM zexcl_receivable AS a
    INNER JOIN ztetmp_ui5_ca    AS b ON a~variant = b~receivables_ex_variant INTO TABLE @lt_trans
         WHERE b~ca_category = @lv_vktyp
      ORDER BY a~main_trans, a~sub_trans.
    IF sy-subrc EQ 0.
      SORT lt_trans BY main_trans sub_trans.
      LOOP AT lt_dfkkop ASSIGNING FIELD-SYMBOL(<fs_dfkkop_filter>).
        READ TABLE lt_trans WITH KEY main_trans = <fs_dfkkop_filter>-hvorg
                                     sub_trans  = <fs_dfkkop_filter>-tvorg TRANSPORTING NO FIELDS BINARY SEARCH.
        IF sy-subrc EQ 0 AND <fs_dfkkop_filter>-psobtyp <> lc_regime_itsa. "ARXK963586
          DELETE lt_dfkkop.
        ENDIF.
      ENDLOOP.
    ENDIF.
  ENDIF.
* If no valid credit items remain after filtering, return functional error
  IF lt_dfkkop IS INITIAL.
    es_payment_spec_error = VALUE #( error_type = lc_functional_error
                                     error_desc = TEXT-002  ).

    es_error_log = VALUE #(  id     = lc_itsa_msgid
                             type   = zcl_safe_constants=>gc_error
                             number = lc_no_credit   ).
    RETURN.
  ENDIF.

*End of changes for ARXK961808

* Set Item table for PS creation
  LOOP AT lt_dfkkop ASSIGNING FIELD-SYMBOL(<fs_dfkkop>).
    lv_tot_amount = lv_tot_amount + <fs_dfkkop>-betrw.

* Set Credit charges*******************
    APPEND INITIAL LINE TO lt_dfkkip_item ASSIGNING FIELD-SYMBOL(<fs_dfkkip_itm>).
    MOVE-CORRESPONDING <fs_dfkkop> TO <fs_dfkkip_itm>.
    <fs_dfkkip_itm>-betrw = <fs_dfkkop>-betrw.

    CLEAR: <fs_dfkkip_itm>-effective_date_of_payment,
           <fs_dfkkip_itm>-due_date,
           <fs_dfkkip_itm>-period_to,
           <fs_dfkkip_itm>-period_from.


    IF <fs_dfkkop>-abrzu NE zcl_safe_constants=>gc_init_date.
      <fs_dfkkip_itm>-period_from = <fs_dfkkop>-abrzu.
    ENDIF.

    IF <fs_dfkkop>-abrzo NE zcl_safe_constants=>gc_init_date.
      <fs_dfkkip_itm>-period_to = <fs_dfkkop>-abrzo.
    ENDIF.

    IF <fs_dfkkop>-faedn NE zcl_safe_constants=>gc_init_date.
      IF <fs_dfkkop>-hvorg EQ zcl_safe_constants=>gc_hvorg_payment.
        <fs_dfkkip_itm>-effective_date_of_payment = <fs_dfkkop>-faedn.
      ELSE.
        <fs_dfkkip_itm>-due_date = <fs_dfkkop>-faedn.
      ENDIF.
    ENDIF.


  ENDLOOP.

* Begin of change for ITSA-R6-Unauthorized agent fix
  IF iv_block_repayment EQ abap_true AND abs( lv_tot_amount ) GT lc_tot_amount."ITSA-R6-Unauthorized agent fix

    es_payment_spec_error-error_type = lc_functional_error.
    es_payment_spec_error-error_desc = TEXT-004.

    es_error_log-id = lc_itsa_msgid.
    es_error_log-type   = zcl_safe_constants=>gc_error.
    es_error_log-number = lc_exceed_10.

    RETURN.
  ENDIF.
* End of change for ITSA-R6-Unauthorized agent fix

** If validated, then create PS for credit charges
  IF lt_dfkkop IS NOT INITIAL AND lv_utr IS NOT INITIAL. " AND iv_acc_num is not initial.
    lv_tot_amount = -1 * lv_tot_amount.
* Call geneic method to create PS.
    CREATE OBJECT lr_proc
      EXPORTING
        iv_pdkey = lv_pdkey.                                     " Number of Payment Specification

    IF lr_proc IS BOUND.
* Determine payment METHOD
      IF gv_pymet IS INITIAL.   " if PS was reversed, use same Paym Method
        lr_proc->determine_payment_method(
          EXPORTING
            iv_validate_bank_details = abap_true
            iv_gpart        = iv_gpart       " Business Partner Number
            iv_vkont        = iv_vkont
            iv_tamount      = lv_tot_amount                        " Amount in Transaction Currency with +/- Sign
            iv_po_invalid   = abap_true
            is_bank_details = is_bank_details                        " PYMET- PO invalid for PS creation
          IMPORTING
            ev_pymet        = DATA(lv_pymet)                       " Payment Method
        ).
      ELSE.
        lv_pymet = gv_pymet.
      ENDIF.

* No payment method found
      IF lv_pymet IS INITIAL.
        es_payment_spec_error-error_type = lc_functional_error.
        es_payment_spec_error-error_desc = TEXT-003.

        es_error_log-id    = lc_itsa_msgid.
        es_error_log-type  = zcl_safe_constants=>gc_error.
        es_error_log-number = lc_pymethod.

        RETURN.
      ENDIF.

      ls_dfkkip_item-pymet = lv_pymet.
      MODIFY  lt_dfkkip_item FROM ls_dfkkip_item TRANSPORTING pymet WHERE pymet = space.

* Set Header details*****************
      ls_dfkkip_grp-gparv = iv_gpart.
      ls_dfkkip_grp-vkonv = iv_vkont.
      ls_dfkkip_grp-pdkey = lv_pdkey.

* Create Payment Specification Repayment ID.
      lr_proc->create_repayment(
        EXPORTING
          is_header_selected  =  ls_dfkkip_grp    " Payment Specification header
          it_item_selected    =  lt_dfkkip_item   " Table type for Credit Charges
          iv_ps_origin        =  lc_ps_ret_org
        IMPORTING
          es_error_log        =  ls_log           " Return Parameter
          ev_pdkey            =  ev_pdkey         " Number of Payment Specification
      ).
      IF ls_log IS NOT INITIAL.
        es_error_log = ls_log.
      ENDIF.
    ENDIF.
  ENDIF.
ENDMETHOD.
