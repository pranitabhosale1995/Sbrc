
import React, { useState, useCallback, useEffect, useRef, useMemo } from "react";
import {
  TextField,
  Button,
  Box,
  Container,
  Paper,
  Typography,
  Grid,
  InputAdornment,
  Card,
  CardContent,
  Alert,
  Snackbar,
  Chip,
} from "@mui/material";
import { DataGrid } from "@mui/x-data-grid";
import CalendarMonthIcon from "@mui/icons-material/CalendarMonth";
import SearchIcon from "@mui/icons-material/Search";
import useApi from "../../hooks/useApi";
import useValidation from "./Validation"; // ✅ added

const INITIAL_FILTERS = {
  fromDate: "",
  toDate: "",
  branchCode: "",
  currency: "",
  cgl: "",
};

const INITIAL_PAGINATION = { page: 0, pageSize: 100 };
// const PAGE_SIZE_OPTIONS = [100, 200, 300, 400];
const TODAY = new Date().toISOString().split("T")[0];



const formatNumber = (value) =>
  value != null
    ? Number(value).toLocaleString("en-IN", {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2,
      })
    : "-";



const buildPayload = (filters) => ({
  fromDate:filters.fromDate,
  toDate: filters.toDate,
  branchCode:filters.branchCode?.trim() || null,
  currency: filters.currency?.trim() || null,
  cgl: filters.cgl?.trim() || null,
});

const mapRowData = (item, pageIndex, index) => ({
  id: item.id ?? `row-${pageIndex}-${index}`,
  ERROR_DATE: item.first_ERROR_DATE || "-",
  BRANCH_CODE: item.branchCode || "-",
  CURRENCY: item.currency || "-",
  CGL: item.cgl || "-",
  CBS_BALANCE: item.cbsBalance ?? 0,
  GL_BALANCE: item.glBalance ?? 0,
  DIFFERENCE_AMOUNT: item.differenceAmount ?? 0,
  DIFF_YESTERDAY: item.diffBwYesterday ?? 0,
  TYPE_VAL: item.type || "-",
  HEAD_VAL: item.head || "-",
});

const DifferenceCell = ({ value }) => (
  <Typography
    variant="body2"
    sx={{
      color: value < 0 ? "error.main" : value > 0 ? "success.main" : "text.secondary",
      fontWeight: 600,
      fontFeatureSettings: '"tnum"',
    }}
  >
    {formatNumber(value)}
  </Typography>
);

const NumericCell = ({ value }) => (
  <Typography
    variant="body2"
    sx={{ fontFeatureSettings: '"tnum"', textAlign: "right", width: "100%" }}
  >
    {formatNumber(value)}
  </Typography>
);

const BalanceRangeSearchScreen = () => {
  const { callApi } = useApi();
  const { validateBranchCode, validateCurrency, validateCGL } = useValidation(); // ✅ added

  const [filters, setFilters] = useState(INITIAL_FILTERS);
  const [rows, setRows] = useState([]);
  const [loading, setLoading] = useState(false);
  const [showTable, setShowTable] = useState(false);
  const [totalElements, setTotalElements] = useState(0);
  const [paginationModel, setPaginationModel] = useState(INITIAL_PAGINATION);
  const [snackbar, setSnackbar] = useState({ open: false, message: "", severity: "error" });
  const [validationErrors, setValidationErrors] = useState({});

  const isFirstRender = useRef(true);

  const validate = useCallback(() => {
    const errors = {};
    if (!filters.fromDate) errors.fromDate = "From Date is required";
    if (!filters.toDate) errors.toDate = "To Date is required";
    if (filters.fromDate && filters.fromDate > TODAY)
      errors.fromDate = "From Date cannot be a future date";
    if (filters.toDate && filters.toDate > TODAY)
      errors.toDate = "To Date cannot be a future date";
    if (filters.fromDate && filters.toDate && filters.fromDate > filters.toDate)
      errors.toDate = "To Date must be after From Date";
    setValidationErrors(errors);
    return Object.keys(errors).length === 0;
  }, [filters.fromDate, filters.toDate]);

  const updateFilter = useCallback((key, value) => {
    setFilters((prev) => ({ ...prev, [key]: value }));
    setValidationErrors((prev) => ({ ...prev, [key]: undefined }));
  }, []);

  const fetchDifferences = useCallback(
    async (pageIndex, pageSize) => {
      setLoading(true);
      try {
        const response = await callApi(
          `/ES/differences/search?page=${pageIndex}&size=${pageSize}`,
          buildPayload(filters),
          "POST"
        );

        const content = response?.data?.content ?? response?.content ?? [];
        const totalCount = response?.data?.totalElements ?? response?.totalElements ?? 0;

        if (!Array.isArray(content)) throw new Error("Unexpected response format.");

        setRows(content.map((item, index) => mapRowData(item, pageIndex, index)));
        setTotalElements(totalCount);

        if (content.length === 0) {
          setSnackbar({ open: true, message: "No records found for the selected filters.", severity: "info" });
        }
      } catch (err) {
        console.error("[BalanceRangeSearch] API Error:", err);
        setRows([]);
        setTotalElements(0);

        const status = err?.response?.status || err?.status;
        let message = "Failed to fetch data. Please try again.";
        if (status === 500) message = "Server error (500). Please contact support or try again later.";
        else if (status === 404) message = "API endpoint not found (404). Please check configuration.";
        else if (status === 401 || status === 403) message = "Unauthorized access. Please login again.";
        else if (err?.message) message = err.message;

        setSnackbar({ open: true, message, severity: "error" });
      } finally {
        setLoading(false);
      }
    },
    [filters, callApi]
  );

  useEffect(() => {
    if (isFirstRender.current) {
      isFirstRender.current = false;
      return;
    }
    if (showTable) {
      fetchDifferences(paginationModel.page, paginationModel.pageSize);
    }
  }, [paginationModel.page, paginationModel.pageSize]); // eslint-disable-line react-hooks/exhaustive-deps

  const handleSearch = useCallback(() => {
    if (!validate()) return;
    setShowTable(true);
    const reset = INITIAL_PAGINATION;
    setPaginationModel(reset);
    fetchDifferences(reset.page, reset.pageSize);
  }, [validate, fetchDifferences]);

  const columns = useMemo(
    () => [
      { field: "ERROR_DATE", headerName: "Error Date", width: 120, headerAlign: "center", align: "center" },
      { field: "BRANCH_CODE", headerName: "Branch", width: 90, headerAlign: "center", align: "center" },
      { field: "CURRENCY", headerName: "Currency", width: 90, headerAlign: "center", align: "center" },
      { field: "CGL", headerName: "CGL", width: 140, headerAlign: "center", align: "center" },
      {
        field: "CBS_BALANCE", headerName: "CBS Balance", width: 160, headerAlign: "right", align: "right",
        renderCell: ({ value }) => <NumericCell value={value} />,
      },
      {
        field: "GL_BALANCE", headerName: "GL Balance", width: 160, headerAlign: "right", align: "right",
        renderCell: ({ value }) => <NumericCell value={value} />,
      },
      {
        field: "DIFFERENCE_AMOUNT", headerName: "Difference Amount", width: 180, headerAlign: "right", align: "right",
        renderCell: ({ value }) => <DifferenceCell value={value} />,
      },
      {
        field: "DIFF_YESTERDAY", headerName: "Diff. Yesterday", width: 160, headerAlign: "right", align: "right",
        renderCell: ({ value }) => <NumericCell value={value} />,
      },
      {
        field: "TYPE_VAL", headerName: "Type", width: 100, headerAlign: "center", align: "center",
        renderCell: ({ value }) =>
          value !== "-" ? (
            <Chip label={value} size="small" variant="outlined" color="primary" />
          ) : (
            <Typography variant="body2" color="text.secondary">-</Typography>
          ),
      },
      { field: "HEAD_VAL", headerName: "Head", width: 130, headerAlign: "center", align: "center" },
    ],
    []
  );

  return (
    <Container maxWidth="xl" sx={{ mt: 4, mb: 4 }}>
   

      <Card sx={{ mb: 4, boxShadow: "0 4px 20px rgba(0,0,0,0.08)", borderRadius: 2 }}>
        <CardContent sx={{ p: 3 }}>
          <Grid container spacing={2} alignItems="flex-start">

            <Grid item xs={12} sm={6} md={2.4}>
              <TextField
              
                fullWidth label="From Date" type="date" size="small"
                value={filters.fromDate}
                onChange={(e) => updateFilter("fromDate", e.target.value)}
                InputLabelProps={{ shrink: true }}
                InputProps={{
                 
                  inputProps: { max: TODAY },
                }}
                onKeyDown={(e)=>e.preventDefault()}
                error={!!validationErrors.fromDate}
                helperText={validationErrors.fromDate}
              />
            </Grid>

            <Grid item xs={12} sm={6} md={2.4}>
              <TextField
                fullWidth label="To Date" type="date" size="small"
                value={filters.toDate}
                onChange={(e) => updateFilter("toDate", e.target.value)}
                InputLabelProps={{ shrink: true }}
                InputProps={{
                 
                  inputProps: { max: TODAY },
                }}
                 onKeyDown={(e)=>e.preventDefault()}
                error={!!validationErrors.toDate}
                helperText={validationErrors.toDate}
              />
            </Grid>

            <Grid item xs={12} sm={4} md={2}>
              <TextField
                fullWidth label="Branch Code" size="small"
                value={filters.branchCode}
                onChange={(e) =>
                  updateFilter(
                    "branchCode",
                    validateBranchCode(e.target.value)
                  )
                }
                inputProps={{ maxLength: 20 }}
              />
            </Grid>

            <Grid item xs={12} sm={4} md={2}>
              <TextField
                fullWidth label="Currency" size="small"
                value={filters.currency}
                 onChange={(e) =>
                  updateFilter(
                    "currency",
                    validateCurrency(e.target.value)
                  )
                }
                inputProps={{ maxLength: 3 }}
              />
            </Grid>

            <Grid item xs={12} sm={4} md={3.2}>
              <TextField
                fullWidth label="CGL" size="small"
                value={filters.cgl}
                 onChange={(e) =>
                  updateFilter("cgl", validateCGL(e.target.value))
                }
                InputProps={{
                  startAdornment: (
                    <InputAdornment position="start">
                      <SearchIcon fontSize="small" color="action" />
                    </InputAdornment>
                  ),
                }}
              />
            </Grid>

            <Grid item xs={12}>
              <Box sx={{ display: "flex", justifyContent: "flex-end" }}>
                <Button
                  variant="contained"
                  onClick={handleSearch}
                  disabled={loading}
                  sx={{ height: 40, borderRadius: 1, textTransform: "none", fontWeight: 600, px: 6, minWidth: 160 }}
                >
                  {loading ? "Searching..." : "Search"}
                </Button>
              </Box>
            </Grid>

          </Grid>
        </CardContent>
      </Card>

      {showTable && (
        <Paper sx={{ borderRadius: 2, overflow: "hidden", boxShadow: "0 4px 20px rgba(0,0,0,0.08)" }}>
          <Box sx={{ px: 2, py: 1.5, borderBottom: "1px solid", borderColor: "divider" }}>
            <Typography variant="subtitle2" color="text.secondary">
              {loading ? "Loading..." : `${totalElements.toLocaleString()} record(s) found`}
            </Typography>
          </Box>
          <DataGrid
            rows={rows}
            columns={columns}
            rowCount={totalElements}
            loading={loading}
            paginationMode="server"
            paginationModel={paginationModel}
            onPaginationModelChange={setPaginationModel}
            // pageSizeOptions={[10, 25, 50, 100]}
            disableRowSelectionOnClick
           
            density="compact"
            sx={{
              border: "none",
              "& .MuiDataGrid-columnHeaders": { backgroundColor: "#f5f5f5", fontWeight: 700, fontSize: "0.8rem" },
              "& .MuiDataGrid-row:hover": { backgroundColor: "#f0f4ff" },
              "& .MuiDataGrid-cell": { fontSize: "0.82rem" },
            }}
          />
        </Paper>
      )}

      <Snackbar
        open={snackbar.open}
        autoHideDuration={4000}
        onClose={() => setSnackbar((prev) => ({ ...prev, open: false }))}
        anchorOrigin={{ vertical: "bottom", horizontal: "center" }}
      >
        <Alert
          severity={snackbar.severity}
          onClose={() => setSnackbar((prev) => ({ ...prev, open: false }))}
          sx={{ width: "100%" }}
        >
          {snackbar.message}
        </Alert>
      </Snackbar>
    </Container>
  );
};

export default BalanceRangeSearchScreen;
