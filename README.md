import React, { useState, useCallback, useEffect } from "react";
import {
  TextField,
  Button,
  Box,
  Container,
  Paper,
  Typography,
  IconButton,
  Grid,
  Card,
  CardContent,
  Divider,
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions,
  InputAdornment,
} from "@mui/material";
import { DataGrid } from "@mui/x-data-grid";
import HistoryIcon from "@mui/icons-material/History";
import CloseIcon from "@mui/icons-material/Close";
import SearchIcon from "@mui/icons-material/Search";
import useApi from "../../hooks/useApi";
import useValidation from "./Validation"; // ✅ added

const BalanceParticularSearchScreen = () => {
  const { callApi } = useApi();
  const { validateBranchCode, validateCurrency, validateCGL } = useValidation(); // ✅ added

  const [filters, setFilters] = useState({
    date: "",
    branchCode: "",
    currency: "",
    cgl: "",
  });

  const [rows, setRows] = useState([]);
  const [loading, setLoading] = useState(false);
  const [totalElements, setTotalElements] = useState(0);
  const [etlDate, setEtlDate] = useState();

  const [paginationModel, setPaginationModel] = useState({
    page: 0,
    pageSize: 10,
  });

  const [openHistory, setOpenHistory] = useState(false);
  const [selectedRow, setSelectedRow] = useState(null);
const [showTable, setShowTable] = useState(false);
  const updateFilter = (key, value) =>
    setFilters((prev) => ({ ...prev, [key]: value }));

  // 1. Fetch Default ETL Date on Page Load
  useEffect(() => {
    const fetchDefaultETLDate = async () => {
      try {
        const response = await callApi("/PS/file/fincore-date", {}, "GET");
        const etlDate = response?.data?.etlDate || response?.date || "";
        if (etlDate) {
          setEtlDate(etlDate);
          updateFilter("date", etlDate);
          fetchData(etlDate, 0, paginationModel.pageSize);
        }
      } catch (err) {
        console.error("ETL Date Error:", err);
      }
    };
    fetchDefaultETLDate();
  }, []);

  // 2. Main Search API Call
  const fetchData = useCallback(
    async (date, pageIndex, pageSize) => {
      if (!date) return;
      setLoading(true);
      try {
        const payload = {
          fromDate: date,
          toDate: date,
          branchCode: filters.branchCode || null,
          currency: filters.currency || null,
          cgl: filters.cgl || null,
        };

        const response = await callApi(
          `/ES/differences/search?page=${pageIndex}&size=${pageSize}`,
          payload,
          "POST"
        );

        const content = response?.data?.content || response?.content || [];
        const mappedRows = content.map((item, index) => ({
          id: item.id || `row-${pageIndex}-${index}`,
          ...item,
          GLCC_VAL: `${item.branchCode} ${item.currency} ${item.cgl}`,
        }));

        setRows(mappedRows);
        setTotalElements(
          response?.data?.totalElements || response?.totalElements || 0
        );
      } catch (err) {
        console.error("Search Fail:", err);
        setRows([]);
      } finally {
        setLoading(false);
      }
    },
    [filters]
  );

  useEffect(() => {
    if (filters.date) {
      fetchData(filters.date, paginationModel.page, paginationModel.pageSize);
    }
  }, [paginationModel.page, paginationModel.pageSize]);

  const columns = [
    { field: "reconRunDate", headerName: "Error Date" },
    { field: "GLCC_VAL", headerName: "GLCC" },
    { field: "cbsBalance", headerName: "CBS Balance", align: "right" },
    { field: "glBalance", headerName: "GL Balance", align: "right" },
    {
      field: "differenceAmount",
      headerName: "Difference",
      align: "right",
      renderCell: (params) => (
        <Typography
          sx={{
            color: params.value < 0 ? "#d32f2f" : "#2e7d32",
            fontWeight: 700,
          }}
        >
          {params.value?.toLocaleString()}
        </Typography>
      ),
    },
    {
      field: "action",
      headerName: "Action",
      sortable: false,
      renderCell: (params) => (
        <IconButton
          color="primary"
          onClick={() => {
            setSelectedRow(params.row);
            setOpenHistory(true);
          }}
        >
          <HistoryIcon />
        </IconButton>
      ),
    },
  ];

  return (
    <Container maxWidth="xl" sx={{ mt: 4, mb: 4 }}>
       <Card sx={{ mb: 4, boxShadow: "0 4px 20px rgba(0,0,0,0.08)", borderRadius: 2 }}>
        <CardContent sx={{ p: 3 }}>
         <Grid container spacing={2} alignItems="flex-start">
            <Grid item xs={12} sm={6} md={2.4}>
              <TextField
                fullWidth
                label="ETL Date"
                type="date"
                size="small"
                value={etlDate}
                onChange={(e) => {
                  setEtlDate(e.target.value);
                  updateFilter("date", e.target.value);
                }}
                InputLabelProps={{ shrink: true ,
                  
                }}
                
                 onKeyDown={(e)=>e.preventDefault()}
              />
            </Grid>

            {/* ✅ Branch Validation */}
            <Grid item xs={12} sm={3} md={2.4}>
              <TextField
                fullWidth
                label="Branch"
                size="small"
                value={filters.branchCode}
                onChange={(e) =>
                  updateFilter(
                    "branchCode",
                    validateBranchCode(e.target.value)
                  )
                }
              />
            </Grid>

            {/* ✅ Currency Validation */}
            <Grid item xs={12} sm={3} md={2}>
              <TextField
                fullWidth
                label="Currency"
                size="small"
                value={filters.currency}
                onChange={(e) =>
                  updateFilter(
                    "currency",
                    validateCurrency(e.target.value)
                  )
                }
              />
            </Grid>

            {/* ✅ CGL Validation */}
            <Grid item xs={12} sm={8} md={2}>
              <TextField
                fullWidth
                label="CGL Number"
                size="small"
                placeholder="Search by CGL..."
                value={filters.cgl}
                onChange={(e) =>
                  updateFilter("cgl", validateCGL(e.target.value))
                }
                InputProps={{
                  startAdornment: (
                    <InputAdornment position="start">
                      <SearchIcon fontSize="small" />
                    </InputAdornment>
                  ),
                }}
              />
            </Grid>

            <Grid item xs={12} sm={4} md={3.2}>
               <Box sx={{ display: "flex", justifyContent: "flex-end" }}>
              <Button
                fullWidth
                variant="contained"
                sx={{
                  bgcolor: "#1a237e",
                  padding: "10px 40px",
                }}
                onClick={() => {
                  setPaginationModel((prev) => ({ ...prev, page: 0 }));
                  fetchData(filters.date, 0, paginationModel.pageSize);
                }}
              >
                Search
              </Button>
              </Box>
            </Grid>
          </Grid>
        </CardContent>
      </Card>
 
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
           pageSizeOptions={[10, 25, 50, 100]}
          disableSelectionOnClick

 density="compact"
            sx={{
              border: "none",
              "& .MuiDataGrid-columnHeaders": { backgroundColor: "#f5f5f5", fontWeight: 700, fontSize: "0.8rem" },
              "& .MuiDataGrid-row:hover": { backgroundColor: "#f0f4ff" },
              "& .MuiDataGrid-cell": { fontSize: "0.82rem" },
            }}

        />
      </Paper>

    
    </Container>
  );
};

export default BalanceParticularSearchScreen;
